# amazon-lambda-window-wine
Is it possible for AWS Lambda to run a Windows binary via Wine?

https://stackoverflow.com/questions/42013641/is-it-possible-for-aws-lambda-to-run-a-windows-binary-via-wine

I compiled a custom wine with minimal dependencies for lambda, compressed it and then put it onto S3.

Then, in the lambda at runtime, I downloaded the archive, extracted it to /tmp and ran it with a custom empty wine prefix.

My test windows executable was 64bit curl.exe.

1. Compile Wine for Lambda
From https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html, I first tried amzn-ami-hvm-2018.03.0.20181129-x86_64-gp2, but it had an older compilation environment and wouldn't configure.

With AMI amzn2-ami-hvm-2.0.20190313-x86_64-gp2 on a t3.2xlarge ec2, I was able to configure and compile. These are the commands I used, references aws-compile and building-wine:

> sudo yum groupinstall "Development Tools"
> mkdir -p ~/wine-dirs/wine-source
> git clone git://source.winehq.org/git/wine.git ~/wine-dirs/wine-source
> cd ~/wine-dirs/wine-source
> ./configure --enable-win64 --without-x --without-freetype --prefix /opt/wine
> make -j8
> sudo mkdir -p /opt/wine
> sudo chown ec2-user.ec2-user /opt/wine/
> make install
> cd /opt/
> tar zcvf ~/wine-64.tar.gz wine/
This was only a 64-bit build. It also had almost no other optional wine dependencies.

2. Reduce the size of the Wine build further
I removed a lot of optional dependencies from the wine build at compilation time, but it was still too big. /tmp is limited to 500MB.

I deleted files in the package subdirectories, including what looked like optional libs, until I got it down to around 300MB uncompressed.

I verified that wine would still run curl.exe after deleting files from the build.

3. Compress it
I created a tar.bz2 of wine and curl with default bz2 options, it ended up around 80MB. The compressed and extracted files together required about 390MB.

That way there is enough room to both download the archive and extract it to /tmp inside the lambda.

> du -h .
290M    ./wine/lib64/wine
292M    ./wine/lib64
276K    ./wine/share/wine
8.0K    ./wine/share/applications
288K    ./wine/share
5.0M    ./wine/curl-7.66.0-win64-mingw/bin
5.0M    ./wine/curl-7.66.0-win64-mingw
12M     ./wine/bin
308M    ./wine
390M    .

> ls
wine wine.tar.bz2
4. Upload wine.tar.bz2 to S3
Create an S3 bucket and upload the wine.tar.bz2 file to it.

5. Create the Lambda
Create an AWS Lambda using the python 3.7 runtime. While this uses a different underlying AMI than what wine was built on above, it still worked.

In the lambda execution role, grant access to the S3 bucket.

RAM: 1024MB. I chose this because lambda CPU power scales with the memory.

Timeout: 1 min

6. Lambda code:
I needed to follow the advice from this question and answer to change the wine prefix inside the lambda. I also turned off the display as it suggested.

e.g.:

handler():
  ... download from S3 to /tmp, cd to /tmp

  subprocess.call(["tar", "-jxvf", "./wine.tar.bz2"])
  
  os.environ['DISPLAY'] = ''
  os.environ['WINEARCH'] = 'win64'
  os.environ['WINEPREFIX'] = '/tmp/wineprefix'
  
  subprocess.call(["./wine/bin/wine64", "./wine/curl-7.66.0-win64-mingw/bin/curl.exe", "http://www.stackoverflow.com"])
successful execution of wine in lambda

Success!
