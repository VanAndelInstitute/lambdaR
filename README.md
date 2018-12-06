# lambdaR
Bringing R as runtime to AWS Lambda

## Overview

AWS Lambda now supports [bring your own runtime](https://aws.amazon.com/blogs/aws/new-for-aws-lambda-use-any-programming-language-and-share-common-components/) via there minimalist runtime API. The basic idea is that you create a layer containing your runtime components, and an executable `bootstrap` script that glues together the task request and your runtime.

Pretty basic in concept, but putting the pieces together for using the R runtime took a little effort.  The steps are described here.

Note that you only need to do this once.  When you have a functional R runtime layer, you can use that layer for any lambda functions you create that require the R runtime.

## Build Environment

We want to build our toolchain in the same environment used by Lambda.  So create an instance (it doesn't take much...t2.small or medium should be fine) based on this AMI: `amzn-ami-hvm-2017.03.1.20170812-x86_64-gp2`.  SSH to that instance and update the AWS client tools (the one that comes with that AMI does not know about Lambda layers):

```
sudo pip install awscli --upgrade --user
```

You will also need the development tools, and while we are at it, an editor:

```
sudo yum groupinstall -y "Development Tools"
sudo yum -y install emacs-nox
```

Lambda copies the layer contents into `/opt`, so will install things there to begin with (otherwise, R will get confused).  You will need to update your .bashrc so that the tools we build can find eachother. Here is how I have mym .bashrc configured (probably some of these lines are superfluous):

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
Make sure you start a new bash session before continuing to pick up these changes.

I like to create a "build" directory and work in there.

```
mkdir ~/build
cd ~/build
```

## Building the dependencies

R has a number of dependencies.  Rather than doing `yum install` and then trying to track down the pieces and put them in `/opt`, we will just build them and install into /opt.

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
```

## Building R

Now we will build R with most options inactivated (because layers must be <50MB zipped).

```
wget https://cran.r-project.org/src/base/R-3/R-3.5.1.tar.gz
tar -zxvf R-3.5.1.tar.gz
cd R-3.5.1
./configure --prefix=/opt/R --enable-R-shlib --with-recommended-packages --with-x=no --with-aqua=no \
	--with-tcltk=no --with-ICU=no --disable-java --disable-openmp --disable-nls --disable-largefile \
	--disable-R-profiling --disable-BLAS-shlib --disable-rpath --with-libpng=no --with-jpeglib=no --with-libtiff=no \
	--with-readline=no
make
sudo make install
cd ..
```

Unfortunately, if we zip the resulting build of R, it is a hair larger than 50MB.  So let's prune a couple things out we don't need.

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

## Copying system dependencies

There are a couple of libraries that were installed when we installed the Development Tools group that R depends on, but won't be there in the Lambda environment.  We will copy these into /opt/lib

```
cp /usr/lib64/libgfortran.so.3 /opt/lib
cp /usr/lib64/libquadmath.so.0 /opt/lib

```

## Bootstrap

We now need to create /opt/bootstrap that Lambda will call to do whatever we want to do with our new runtime.  The bootstrap file below parses on the handler (e.g. "test.handler", reformats that to the name of an R script (.e.g "test.r") that should be provided along with your Lambda function (you can upload a zip file with one or more scripts in it when you creat the Lambda function), and then executes that script via `/opt/R/bin/Rscript`. It also retrieves metadata about the request, and sends the response back to let Lambda know it is finished. 

```
#!/bin/sh

# uncomment the following to stop execution on first error
#set -euo pipefail

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


## Package it up

Due to size constraints, we need to put R an its dependencies in separate layers, so we create two zipfiles (moving the aws folder out of the way as we do so)

```
sudo -s
cd /opt
zip -r ../r.zip R
mv R ../
mv aws ../
zip -r ../subsystem.zip *
mv ../R ./
mv ../aws ./

```

Assuming you have already run `aws configure` and entered your credentials, we can now push these to lambda layers.

```
aws lambda publish-layer-version --layer-name subsystem --zip-file fileb://../subsystem.zip
aws lambda publish-layer-version --layer-name R --zip-file fileb://../r.zip

```

You can also push the zip files to S3 for future reference if you desire:

```
aws s3 cp ../subsystem.zip s3://YOURBUCKET/subsystem.zip
aws s3 cp ../r.zip s3://YOURBUCKET/r.zip

```

## Done!

You are now all set to use the R runtime for your lambda functions.  Creata new function, and add your two new layers to it. You will also need to upload an R script to do whatever you want to do with R.  For example, you could create a file `test.r` with this in it:

```
cat("Hello from planet lambdar!")
```

Then zip it up and upload it into your Lambda function.  For the handler, you would specify "test.handler".  The bootstrap script above will parse out the "test" part and execute your "test.r" script (which gets installed in /var/task). 
