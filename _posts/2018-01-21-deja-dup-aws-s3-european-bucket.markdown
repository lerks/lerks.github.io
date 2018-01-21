---
layout: post
title:  "Use a European AWS S3 bucket in Déjà Dup"
date:   2018-01-21 17:00:00 +0100
---

On my [Arch Linux](https://www.archlinux.org/) machine I use [Déjà Dup](https://launchpad.net/deja-dup) to backup my files on the cloud. I chose that program because it offers a simple graphical interface to select which files, where and how often to backup, because it takes care of scheduling backups without forcing me to use cron & co., and because it came pre-installed. I chose to store my backups on the cloud because I wanted them to be cold, off-site and available from nearly anywhere (supposing that anywhere has internet access) and I wanted minimal setup and maintenance effort.

I decided to store them on [Amazon Web Services](https://aws.amazon.com/)' [Simple Storage Service](https://aws.amazon.com/s3/) (AWS S3), mainly because it was the best supported backend when I started using Déjà Dup (I remember I had issues with Google Drive)[^irony]. After creating an AWS account I gave Déjà Dup the credentials and had it finish the setup for me, which consisted essentially in creating a bucket (i.e., a "disk" on S3). By default this resulted in the bucket being created in Amazon's `us-east-1` datacenter, in Virginia (United States). I recently found out that Amazon now has a datacenter in Paris (France) as well, and I decided I wanted to have my backups stored there, to get better bandwidth[^privacy].

[^irony]: Ironically, while writing this post I found out that now Déjà Dup supports and recommends Google Drive and deprecates AWS S3.
[^privacy]: And, as [someone on the internet](https://bugzilla.redhat.com/show_bug.cgi?id=1404866#c5) pointed out, also to benefit from the EU's privacy protection laws, even though I locally encrypt my data before sending it to AWS S3. This is not supported by Déjà Dup out-of-the-box, but I managed to get it to work by patching the source code, similar to how packages from the [Arch User Repository](https://aur.archlinux.org/) (AUR) are built.

I first downloaded the files used to build the official `deja-dup` package (they can be found [here](https://git.archlinux.org/svntogit/community.git/tree/trunk?h=packages/deja-dup): they include the `PKGBUILD` file and possibly other ones, for example patches) and put them in a new directory. I then added the following patch, in a file named `fix-s3.patch`:
```
--- a/libdeja/BackendS3.vala
+++ b/libdeja/BackendS3.vala
@@ -77,7 +77,7 @@ public class BackendS3 : Backend
     }

     var folder = get_folder_key(settings, S3_FOLDER_KEY);
-    return "s3+http://%s/%s".printf(bucket, folder);
+    return "s3://s3.eu-west-3.amazonaws.com/%s/%s".printf(bucket, folder);
   }

   public bool bump_bucket() {
```
This tells Déjà Dup to use a different URL schema when connecting to AWS S3: instead of `s3+http://{bucket}/{folder}` it will use `s3://s3.{region}.amazonaws.com/{bucket}/{folder}`. Observe that the region, in my case `eu-west-3`, is hardcoded. This means that Déjà Dup will only ever try to access buckets in that region and that if I ever want to change region I have to update that patch and rebuild Déjà Dup. It also means that if you are following these instructions and if your bucket is in a datacenter other than Paris you will have to change that value to the appropriate one (you can find the region codes [here](https://docs.aws.amazon.com/general/latest/gr/rande.html)).

I then modified the `PKGBUILD` file to have it apply the above patch before building, as follows:
```
--- a/PKGBUILD
+++ b/PKGBUILD
@@ -13,17 +13,19 @@
 optdepends=('gnome-keyring: save passwords'
             'nautilus: backup extension'
             'python2-boto: Amazon S3 and Google Cloud Storage backend')
-source=(https://launchpad.net/$pkgname/${pkgver%.*}/$pkgver/+download/$pkgname-$pkgver.tar.xz{,.asc}
-        fix-crash-on-restore.patch)
+source=(https://launchpad.net/$pkgname/${pkgver%.*}/$pkgver/+download/$pkgname-$pkgver.tar.xz
+        fix-crash-on-restore.patch
+        fix-s3.patch)
 validpgpkeys=('A3A5C2FC56AE7341D308D8571B50ECA373F3F233') # Michael Terry
 sha256sums=('2c433a334bcead16f92a98914d36fbf6911cd11dcc75bc8163cefa73fff2fc22'
-            'SKIP'
-            '9b3c66d83325874d3ebe394240962e8d88bc2dc0a48d0550cb4f503f2d8d2554')
+            '9b3c66d83325874d3ebe394240962e8d88bc2dc0a48d0550cb4f503f2d8d2554'
+            '9ac4773c440f0054e34c7ab86bb60f40fce2d1e658457200face6ed884e799ad')

 prepare() {
   mkdir build
   cd $pkgname-$pkgver
   patch -Np1 -i ../fix-crash-on-restore.patch
+  patch -Np1 -i ../fix-s3.patch
 }

 build() {
```
I also had to remove the verification of the downloaded sources as I was getting errors (I probably didn't have the key they were signed with in my keyring).

After that I just ran the usual `makepkg -si` to build and install.

Finally, I went to the [AWS console](https://console.aws.amazon.com/console/home), manually deleted the previous bucket and created a new one with the same name in the new region. I then triggered a manual backup on Déjà Dup to make sure everything worked properly (this created a new full fresh backup).

Since I was there...
---

As I was already patching software, I decided to do the same with [Duplicity](http://duplicity.nongnu.org/), which is the command-line tool that Déjà Dup uses under the hood. There are some useful parameters that can be set when using Duplicity from the command line but that AFAIK are not set by Déjà Dup when it invokes Duplicity. I worked around that by changing the default value that Duplicity uses for those configuration options.

*[AFAIK]: As Far As I Know

You can imagine how this goes. I downloaded the package files from [here](https://git.archlinux.org/svntogit/community.git/tree/trunk?h=packages/duplicity) and I created a file `fix-s3.patch` with the following content:
```
--- a/duplicity/globals.py
+++ b/duplicity/globals.py
@@ -176,11 +176,11 @@ async_concurrency = 0
 # Whether to use "new-style" subdomain addressing for S3 buckets. Such
 # use is not backwards-compatible with upper-case buckets, or buckets
 # that are otherwise not expressable in a valid hostname.
-s3_use_new_style = False
+s3_use_new_style = True

 # Whether to create European buckets (sorry, hard-coded to only
 # support european for now).
-s3_european_buckets = False
+s3_european_buckets = True

 # File owner uid keeps number from tar file. Like same option in GNU tar.
 numeric_owner = False
@@ -193,10 +193,10 @@ s3_unencrypted_connection = False
 s3_use_rrs = False

 # Whether to use S3 Infrequent Access Storage
-s3_use_ia = False
+s3_use_ia = True

 # True if we should use boto multiprocessing version
-s3_use_multiprocessing = False
+s3_use_multiprocessing = True

 # Chunk size used for S3 multipart uploads.The number of parallel uploads to
 # S3 be given by chunk size / volume size. Use this to maximize the use of
```
This mainly changes some flags that control how buckets are accessed and created (I'm not even sure these are actually necessary). It also changes the class the backup files are stored as: from standard to infrequent access (IA), which is cheaper. Finally it enables multiprocessing in the underlying library, which allows multiple files to be uploaded in parallel and helps in using all available bandwidth.

I then patched the `PKGBUILD` file as follows:
```
--- a/PKGBUILD
+++ b/PKGBUILD
@@ -18,11 +18,17 @@
             'gvfs: GIO backend'
             'python2-gdata: Google Docs backend'
             'rsync: rsync backend')
-source=("https://launchpad.net/$pkgname/0.7-series/${pkgver}/+download/$pkgname-$pkgver.tar.gz"{,.sig})
+source=("https://launchpad.net/$pkgname/0.7-series/${pkgver}/+download/$pkgname-$pkgver.tar.gz"
+        "fix-s3.patch")
 md5sums=('d106f93627973026707fdcaf37a578bd'
-         'SKIP')
+         '041c781c5462f7b15c8a9325e7b392f3')
 validpgpkeys=('9D95920CED4A8D5F8B086A9F8B6F8FF4E654E600')

+prepare() {
+  cd "${srcdir}/${pkgname}-${pkgver}"
+  patch -Np1 -i ../fix-s3.patch
+}
+
 build() {
   cd "${srcdir}/${pkgname}-${pkgver}"
```
and ran `makepkg -si`.

One last tip
---

Duplicity keeps a local cached copy of the backup's metadata and syncs it with the remote storage at every invocation. When I first ran Duplicity it told me the cache was empty and started downloading a new copy of it. That was because by default it looks for the cache in `~/.cache/duplicity`, whereas when it is invoked by Déjà Dup it is told to use a cache located in `~/.cache/deja-dup`. To have it use that same cache when invoked manually just add `--archive-dir ~/.cache/deja-dup` to the command line.
