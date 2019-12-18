## Overview

AWS Lambda now supports [bring your own runtime](https://aws.amazon.com/blogs/aws/new-for-aws-lambda-use-any-programming-language-and-share-common-components/) via there minimalist runtime API. The basic idea is that you create a layer containing your runtime components, and an executable `bootstrap` script that glues together the task request and your runtime.

Pretty basic in concept, but putting the pieces together for using the R runtime took a little effort.  The steps are described here.

Note that you only need to do this once.  When you have a functional R runtime layer, you can use that layer for any lambda functions you create that require the R runtime.

## Build Environment

We want to build our toolchain in the same environment used by Lambda.  So create an instance (it doesn't take much...t2.small or medium should be fine) based on the AMI used by Lambda, which as of this writing is this one: `amzn-ami-hvm-2018.03.0.20181129-x86_64-gp2`.  SSH to that instance and update the AWS client tools (the one that comes with that AMI does not know about Lambda layers):

```
sudo pip install awscli --upgrade --user
```

You will also need the development tools, and, while we are at it, an editor:

```
sudo yum groupinstall -y "Development Tools"
sudo yum -y install emacs-nox
```

Lambda copies the layer contents into `/opt`, so we will install things there to begin with (otherwise, R will get confused).  You will need to update your .bashrc so that the tools we build can find eachother. You should do this as root 
as we will need these variables available later when installing R packages as root. Here is how I have my .bashrc configured (probably some of these lines are superfluous):

```
export PATH="$PATH:/opt/bin:/opt/lib:/opt/R/bin"
export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/opt/lib"
export LIBRARY_PATH="$LD_LIBRARY_PATH:/opt/lib"
export LD_RUN_PATH="$LD_RUN_PATH:/opt/lib"
export C_INCLUDE_PATH="$C_INCLUDE_PATH:/opt/include"
export CPLUS_INCLUDE_PATH="$CPLUS_INCLUDE_PATH:/opt/include"
export CPATH="$CPATH:/opt/include"
export LDFLAGS="-I/opt/lib"
```
Make sure you start a new bash session before continuing, so that these environment variables are loaded.

Finally, run `aws configure` to enter your AWS credentials so that you can push lambda layers up to AWS below. Note that 
the credentials you provide must be for an IAM user that has the AWSLambdaFullAccess permissions (you can actually be more granular than that, the key permission is lambda:PublishLayerVersion).

## Building the dependencies

I like to create a "build" directory and work in there.

```
mkdir ~/build
cd ~/build
```

R has a number of dependencies.  Rather than doing `yum install` and then trying to track down the pieces and put them in `/opt`, we will just build them and install into /opt. Note that each tools needs a unique combination of config flags and make commands to generate the desired shared libraries with necessary features installed in the /opt file tree.

```
# openssl
wget https://www.openssl.org/source/openssl-1.0.2q.tar.gz
tar -zxvf openssl-1.0.2q.tar.gz
cd openssl-1.0.2q
./config --prefix=/opt shared
make 
sudo make install
cd ..

# curl
wget https://curl.haxx.se/download/curl-7.62.0.tar.gz
tar -zxvf curl-7.62.0.tar.gz
cd curl-7.62.0
./configure --prefix=/opt --with-ssl
make 
sudo make install
cd ..

# bzip2
wget -O bzip2-1.0.6.tar.gz https://sourceforge.net/projects/bzip2/files/bzip2-1.0.6.tar.gz/download
tar -zxvf bzip2-1.0.6.tar.gz
cd bzip2-1.0.6
make -f Makefile-libbz2_so
sudo make install PREFIX=/opt
cd ..

# xz
wget https://tukaani.org/xz/xz-5.2.4.tar.gz
tar -zxvf xz-5.2.4.tar.gz
cd xz-5.2.4
./configure --prefix=/opt
make 
sudo make install
cd ..

# pcre
wget https://ftp.pcre.org/pub/pcre/pcre-8.42.tar.gz
tar -zxvf pcre-8.42.tar.gz
cd pcre-8.42
./configure --prefix=/opt --enable-utf8 --enable-unicode-properties
make 
sudo make install
cd ..

# libxml2
wget ftp://xmlsoft.org/libxml2/libxml2-2.9.9.tar.gz
tar -zxvf libxml2-2.9.9.tar.gz
cd libxml2-2.9.9
./configure --prefix=/opt 
make 
sudo make install
cd ..

```

## Copying system dependencies

There are a couple of libraries that were installed when we installed the Development Tools group that R depends on, but won't be there in the Lambda environment.  We will copy these into /opt/lib

```
cp /usr/lib64/libgfortran.so.3 /opt/lib
cp /usr/lib64/libquadmath.so.0 /opt/lib

```

## Building R

Now we will build R with most options inactivated (because layers must be <50MB zipped). You can tweak these options to suit your needs, but you may need to do additional pruning after building to get your zip file under 50MB.  Hopefully, larger layers will be supported by AWS soon.

Note that after issuing the "make" command, it is a good time to go get a cup of coffee.

```
wget https://cran.r-project.org/src/base/R-3/R-3.6.1.tar.gz
tar -zxvf R-3.6.1.tar.gz
cd R-3.6.1
./configure --prefix=/opt/R --enable-R-shlib --with-recommended-packages --with-x=no --with-aqua=no \
	--with-tcltk=no --with-ICU=no --disable-java --disable-openmp --disable-nls --disable-largefile \
	--disable-R-profiling --disable-BLAS-shlib --disable-rpath --with-libpng=no --with-jpeglib=no --with-libtiff=no \
	--with-readline=no
make
sudo make install
cd ..
```
Let's prune a couple things out we don't need.

```
sudo -s
cd /opt
rm -rf /opt/R/lib64/R/library/tcltk
mv /opt/R/lib64/R/library/translations/en* /opt/R/lib64/R/library/
mv /opt/R/lib64/R/library/translations/DESCRIPTION /opt/R/lib64/R/library/
rm -rf /opt/R/lib64/R/library/translations/*
mv /opt/R/lib64/R/library/en* /opt/R/lib64/R/library/translations/
mv /opt/R/lib64/R/library/DESCRIPTION /opt/R/lib64/R/library/translations/
```

## Additional libraries

We will add the aws.s3 library and its dependencies (unlike the botor package, aws.s3 does not rely on Python. The paws package is another good option.) However, 
to stay under size limits, we are going to want to install additional libraries under a separate directory.

```
sudo -s
mkdir -p /opt/local/lib/R/site-library
sudo /opt/R/bin/R
install.packages("aws.s3", lib="/opt/local/lib/R/site-library")
q()
```

To allow use of these packages "out of the box", we need to edit `/opt/R/lib64/R/etc/Renviron` to add:

```
R_LIBS_USER=${R_LIBS_USER-'/opt/local/lib/R/site-library'}
```

## Bootstrap

We now need to create `/opt/bootstrap` that Lambda will call to kick off our new runtime.  While not complicated, the `bootstrap` script is the magic sauce that allows a wide range of runtimes to be shoe-horned into the Lambda architecture.  The `bootstrap` file below parses the handler specified by your Lambda functions (e.g. "test.handler"), reformats that to the name of an R script (e.g. "test.r") and then executes that script via `/opt/R/bin/Rscript`. (Note that the corresponding script must be provided along with your Lambda function.  When you create a new Lambda function using the AWS web console, you are prompted for a zip file containing this script.) It also retrieves metadata about the request, and sends the response back to let Lambda know it is finished. 

```
#!/bin/sh

# uncomment the following to stop execution on first error
#set -euo pipefail

# we will add python dependencies here later
export PYTHONPATH=/opt/python

# Initialization - load function handler
SCRIPT=$LAMBDA_TASK_ROOT/$(echo "$_HANDLER" | cut -d. -f1).r

while true
do
    # Get an event
    HEADERS="$(mktemp)"
    EVENT_DATA=$(curl -sS -LD "$HEADERS" -X GET "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/next")
    REQUEST_ID=$(grep -Fi Lambda-Runtime-Aws-Request-Id "$HEADERS" | tr -d '[:space:]' | cut -d: -f2)

    echo "About to run $SCRIPT"
    # Execute the handler function from the script
    RESULT=$(mktemp)
    /opt/R/bin/Rscript $SCRIPT
    RESPONSE="OK"

    # Send the response
    curl -X POST "http://${AWS_LAMBDA_RUNTIME_API}/2018-06-01/runtime/invocation/$REQUEST_ID/response"  -d "$RESPONSE"
done
```
Save this as /opt/bootstrap, and don't forget to `chmod 755` it to make it executable.

Obviously, we could get a lot more fancy by sending the script as a payload with the function request, passing environment variables, sending results to S3, etc. etc.

## Package it up

Due to size constraints, we need to put R, our extra libraries, and its dependencies in separate layers, so we create two zipfiles (moving the aws folder out of the way as we do so)

```
sudo -s
cd /opt
chmod 755 bootstrap
zip -r ../r.zip R
mv R ../
mv aws ../
zip -r ../r_lib.zip local
mv local ../local_hold
zip -r ../subsystem.zip *
mv ../R ./
mv ../aws ./
mv ../local_hold local
```

## Boto layer (DEPRECATED)

This is no longer necessary, but I will leave this here in case I need to refer to it in the future:

Although boto3 is allegedly installed on the AMI used by Lambda functions, I can't seem to find it. For example, `python -c "import boto3"` fails. And 
loading the `botor` R package also fails. So for now, I create another layer to contain python dependencies.

```
mkdir python
cd python
pip install boto3 -t ./
cd ..
zip -r python_lib.zip python
```

## Push layers to AWS.
Assuming you have already run `aws configure` and entered your credentials, we can now push these to lambda layers.

```
aws lambda publish-layer-version --layer-name subsystem --zip-file fileb://../subsystem.zip
aws lambda publish-layer-version --layer-name R --zip-file fileb://../r.zip
aws lambda publish-layer-version --layer-name r_lib --zip-file fileb://../r_lib.zip

```

You can then create a new lambda function through the AWS web console, adding the three layers above to it.

You will need a function to call. For starters, we can try:

```
library(aws.s3)
cat(bucket_exists("jlab-test"))
```

Save that as test.r, and zip it

```
zip test.zip test.r
```

You can then upload that as your function code (make sure to set the handler name as "test.handler")/

Make sure to allocate enough memory to the lambda function. Failure to do so will result in failure with a time out. The clue will be the "Max Memory Used:" will be the same as "Memory Size" in the log output. Also make sure to allocate enough time. The default 3 seconds is not enough to load up the R runtime and execute even a basic test script. 
