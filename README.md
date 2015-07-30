## Contents

This repository contains release builds (distributions) of fpCalc for the following platforms:

- android x86
- android armeabi
- android armeabi-v7a
- windows (32bit)
- linux (x86)

All libraries and executables are *statically-linked* to the degree possible, so they *should not* depend on the installation of any additional shared libraries.

*Everybody can deal with gzip files, right?*

All that is needed is to download (and unzip) the given file and place it where you want it on your system, set it’s executable permissions if on linux, and run it from a command line, or by calling the executable from a program, however you might be familiar with calling executable programs from the language of your choice.

The linux executable *should* work on any version of linux that support the x86 ABI but please note that it was *built, and **only** tested*, on Ubuntu 12.04.

The windows 32bit executable *should* be “good enough” for 64 bit windows usage.

## Examples

The **executables** can be called directly from a command line … a DOS box on windows, or a terminal window in linux, or they can be called from programming languages.  A few examples of how to call them from programming languages are provided below.

The **JNI libraries** can be called directly from your Android Java source code. An example of how to do so is given below.

Parsing the text results is left as an exercise to the reader, but basically the result is “\n” delimited lines consisting of lval=rval pairs, one of which could, for example, be the “fingerprint” of the audio file.

### Perl (windows executable)

So, for example, if you downloaded, unzipped, and copied the windows 32 bit executable to a directory called C:\blah\stuff, you could call it on an mp3 file, and get the results in Perl, by including the following code in your Perl script:

```perl
    my $text = `c:/blah/stuff/fpcalc.exe  some_audio_filename.mp3`;
```
### Java (android executable)

If you download, unzip, and copy the android executable to /system/bin on your android device, and set its permissions, perhaps by first unzipping it on your working platform, say, to a file called C:\blah\stuff\fpcalc, and then “pushing” it to the android device with the following commands:

    adb remount
    adb push C:\blah\stuff\fpcalc /system/bin
    adb shell chmod 755 /system/bin/fpcalc

Then you can ‘simply’ call it from your Android java code by invoking a process to call the executable and capture its output stream, perhaps as shown below:

```java
    public static String call_exe(String[] args)
    {
        String result = null;

        try
        {
            Process process = new ProcessBuilder()
                    .command(args)
                    .redirectErrorStream(true)
                    .start();

            InputStream stdin = process.getInputStream();
            OutputStream stdout = process.getOutputStream();
            InputStreamReader isr = new InputStreamReader(stdin);
            BufferedReader br = new BufferedReader(isr);

            String line = null;
            while ((line = br.readLine()) != null)
            {
                if (result == null) result = "";
                result += line + "\n";
            }

            int exit_value = process.waitFor();
            if (exit_value != 0) result += "process_exit_value=" + exit_value + "\n";
            process.destroy();
        }
        catch (Exception e)
        {
            if (result != null) result += "exception=" + e + "\n";
        }
        return result;
    }

    // from your routine
    ...
    String[] args = { “/system/bin/fpcalc”, “some_audio_filename.mp3” };
    String results = call_exe(args);
    ...
```

Of course, this approach is hugely impractical for a real android application, inasmuch as the installation of /system/bin/fpcalc requires **root permissions**, and your application would have to do the installation, which is in itself a complicated process.  You would either have to carry all the executables in an APK, and detect the platform, and install the executable, or have separate per-platform APKs.  And it would require super-user (root) permissions!!

This whole idea of installing and calling system executables is completely outside of the android-app/playstore architecture, and thus this approach turns out to be extremely complicated if all you want to do is simply call fpCalc on some mp3 files from a simple (or complex) android application.

### Java (jni library from Android Studio)

Finally, maybe, there is an easier solution for calling fpcalc from generic Android Apps by utilizing libfpcalc.so as provided here.

All that is required in your java code is to (a) **declare** the native fpCalc() method, (b) **load** the libfpcalc.so shared library, and (c) **call** fpCalc() from your java code.

The tricky part is loading the shared library. There are two methods for loading a library in Android:

    System.loadLibrary() - uses the "short" name and search paths
    System.load() - uses fully qualified filename

If you call System.load() you explicitly specify the library (i.e. /data/local/tmp/libfpcalc.so).  If you call System.load("fpcalc") it will look for libfpcalc.so on a variety of paths, including, for example, /system/lib.  In these two cases you are responsible for installing the library and setting it's file permissions, etc.

An interesting alternative is available (at least) in **Android Studio**. Android Studio supports the "automatic" installation of "local" shared libraries within your APK file.

You simply **create a folder called "jniLibs"** as a sibling of the **"java"** folder that contains the source code for the class that will be calling fpCalc, and within that jniLibs folder you create (one or more) subfolder(s) named **x86**, **armeabi**, or **armeabi-v7a**, and place the appropriate libfpcalc.so file(s) in that (those) folders.

So if your java class MainActivity, was in a folder structure like this:

    +-- blah
        +-- app
            +-- java
                +-- MainActivity.java

You would add jniLibs and one or more of the following subfolders, and place the appropriate libfpcalc.so files in those subfolders:

    +-- blah
        +-- app
            +-- java
                + MainActivity.java
            +-- jniLibs
                +-- armeabi
                    + libfpcalc.so
                +-- armeabi-v7a
                    + libfpcalc.so
                +-- x86
                    + libfpcalc.so

If you package libfpcalc like this, then when you build your APK in Android Studio it will *automatically* include, and *install*, the correct shared library for your client's platform, and all you have to do is call System.loadLibrary("fpcalc").


```java
    public native String fpCalc(String[] args);
        // declare this in your java class (in this example, MainActivity)

    void test_fpcalc()
    {
        // load the library.
        //
        // loadLibrary can be called from a static, or in your onCreate()
        // method, or wherever, just as long as it's called before
        // the native fpCalc() method.
        //
        // The try/catch should not be needed, but is here
        // in case there was a problem building the libraries.
        //
        // You can also use System.load() with a fully qualified filename.

        try
        {
            System.loadLibrary("fpcalc");
        }
        catch (UnsatisfiedLinkError e)
        {
            System.out.print("Could not load library libfpcalc.so : " + e);
            return;
        }

        // call fpCalc with whataver arguments you want …

        String[] args = {"-md5", "some_filename.mp3"};
        String result = fpCalc(args);
        System.out.print("result = " + result + "\n");

        // parse the result string, for example, for “FINGERPRINT=XXXXX\n”
    }
```

This method, of using the JNI libraries has the advantage that your app does not have to install /system/bin/fpcalc, which in turn means it does not need harsh superuser (root) access priviledges, and all the complications that go with installing such an executable on your client’s android machine.

And once again, parsing the results of the returned String are left as an exercise to the reader.

### JNI_OnLoad()

By the way, these JNI libraries take advantage of a *novel approach* to linking JNI libraries to java classes that **DOES NOT** require the jni library to be built *knowing* what java class it will be linked to (unlike *all other examples of building JNI libraries* that I could find on the net at the time of this writing).  All other examples (approaches to building and linking JNI libraries) would have required (a) me to embed a **constant java** class name in the library's *C/C++ source code*, and for (b) *you* to create/import a *specific Java class* into your program, whose name exactly matches that constant. I found the idea of creating a library that could only be linked to a specific class distateful, so I implemented a different way of "linking" it in JNI_OnLoad().

Please see the phorton1/chromaprint repository, and examples/jniutils.c specifically, for more information on this approach to implementing JNI_OnLoad() so that your C source code doesn’t have to know what java class it will be linked to!

## Additional (new) fpCalc features

These builds support the features and command line parameters found in fpCalc as of the chromaprint v1.2 tip version available on this date (July 23, 2015), including:

    -length SECS
    -version
    -hash
    -raw
    -set CHROMAPRINT_OPTION=VALUE
    -algo NAME

To this set, I have added the following options (as well as beefing up the –version option):

    -md5
    -stream_md5
    -ints
    -av_log LEVEL (default = AV_LOG_ERROR = 16)

Here are the details about my modifications and these additional features

### -version
Beefed up, compared to chromaprint/fpcalc 1.2,  to return ffmpeg and library versions, build and ffmpeg configuration, which can be important in establishing a reference fingerprint.
Example output:

    fpcalc version 1.2.0
    ffmpeg_version "0.9"
    libavutil_version=51.32.0
    libavcodec_version=53.42.0
    libavformat_version=53.24.0
    libswresample_version=0.5.0
    ffmpeg_configuration="--prefix=/home/ubuntu/src/fpcalc/ffmpeg/_linux/host/build --disable-pthreads --bindir=/home/ubuntu/src/fpcalc/ffmpeg/_linux/host/bin --libdir=/home/ubuntu/src/fpcalc/ffmpeg/_linux/host/lib --incdir=/home/ubuntu/src/fpcalc/ffmpeg/_linux/host/include --shlibdir=/home/ubuntu/src/fpcalc/ffmpeg/_linux/host/bin --extra-cflags=-Os --strip=/home/ubuntu/src/fpcalc/ffmpeg/multi-strip --disable-symlink-make --enable-static --disable-shared --enable-memalign-hack --enable-debug --disable-avdevice --disable-avfilter --disable-swscale --disable-ffmpeg --disable-ffplay --disable-ffserver --disable-network --disable-muxers --disable-demuxers --enable-rdft --enable-demuxer=aac --enable-demuxer=ac3 --enable-demuxer=ape --enable-demuxer=asf --enable-demuxer=flac --enable-demuxer=matroska_audio --enable-demuxer=mp3 --enable-demuxer=mpc --enable-demuxer=mov --enable-demuxer=mpc8 --enable-demuxer=ogg --enable-demuxer=tta --enable-demuxer=wav --enable-demuxer=wv --disable-bsfs --disable-filters --disable-parsers --enable-parser=aac --enable-parser=ac3 --enable-parser=mpegaudio --disable-protocols --enable-protocol=file --disable-indevs --disable-outdevs --disable-encoders --disable-decoders --enable-decoder=aac --enable-decoder=ac3 --enable-decoder=alac --enable-decoder=ape --enable-decoder=flac --enable-decoder=mp1 --enable-decoder=mp2 --enable-decoder=mp3 --enable-decoder=mpc7 --enable-decoder=mpc8 --enable-decoder=tta --enable-decoder=vorbis --enable-decoder=wavpack --enable-decoder=wmav1 --enable-decoder=wmav2 --enable-decoder=pcm_alaw --enable-decoder=pcm_dvd --enable-decoder=pcm_f32be --enable-decoder=pcm_f32le --enable-decoder=pcm_f64be --enable-decoder=pcm_f64le --enable-decoder=pcm_s16be --enable-decoder=pcm_s16le --enable-decoder=pcm_s16le_planar --enable-decoder=pcm_s24be --enable-decoder=pcm_daud --enable-decoder=pcm_s24le --enable-decoder=pcm_s32be --enable-decoder=pcm_s32le --enable-decoder=pcm_s8 --enable-decoder=pcm_u16be --enable-decoder=pcm_u16le --enable-decoder=pcm_u24be --enable-decoder=pcm_u24le --enable-decoder=rawvideo"

### -md5
Returns **FINGERPRINT_MD5=** the MD5 hash of the text fingerprint

### -stream_md5
Returns **STREAM_MD5=** MD5 hash of the whole audio “stream”

The idea here is to produce an identifier that identifies the audio stream within the given audio file, independent of what platform you might be running on, and/or any changes to the filename and/or tags within that audio file.

So even if a user changes a filename, and/or the tags for an audio (i.e. mp3) file, theoretically you can still identify the stream within the audio file itself.  This could be used for instance, for maintaining a listening history across different platforms, or for otherwise correlating information to the stream, regardless of the filename or tags.

Note that this is different than the text FINGERPRINT itself and/or the FINGERPRINT_INTS (raw) which are “allowed” to be slightly different across platforms *as long as they compare within a certain threshold*, as defined elsewhere.  The **STREAM_MD5** can be used, as is, for exact matches across platforms.

For a given audio file, all of the builds that I present here as “release” builds will (should) return the same STREAM_MD5, regardless of what platform you run fpCalc on.  Note however, that all of the above platforms are little endian.   A little work (byte flipping) will be needed on the implementation when I, or whoever, implements a bigendian release version of fpcalc, i.e. for OSX.

### -ints
Returns **FINGERPRINT_INTS=** the comma delimited raw fingerprint integers just as –raw does.

The difference is that –ints returns FINGERPRINT_INTS *in addition to* the text FINGERPRINT, whereas -raw returns FINGERPRINT=comma_delimited_integers if called with –raw and FINGERPRINT=text if not.

### -av_log LEVEL
Chromaprint/fpcalc v1.2 invariantly sets ffmpeg’s AV_LOG_LEVEL to AV_LOG_ERROR (which is integer 16).

This option allows you to set the log level to different values.

Try “-av_log 50” for example, to see the AV_LOG_VERBOSE messages in ffmpeg.

## Acknowledgments and Licenses
This work is totally dependent on the ffmpeg and chromaprint open-source projects.

Credits to **Lukas Lalinsky**, **Michael Neidermeyr**, the shoulders they stand on, and the many people who have stood on their shoulders ... especially those that have worked long and hard on the ffmpeg.org project, without whom which I would not even been able to *think of* this project.

The executables and libraries above are released under the GNU General Public License version 2, in accordance with the ffmpeg and chromaprint licenses.  Please see 'COPYING.GPLv2' and the chromaprint and ffmpeg repositories for more information.


## Source Code and Building
Please see the phorton1/chromaprint repository for details on re-creating these release builds of fpCalc.

### Notes on Versioning

These executables and librararies have been built expressely to work as closely as possible to the "official" acousticID fpcalc windows release of fpcalc.exe, of November 23, 2013 as available at http://acoustid.org.

These executables and libraries are based strictly on **ffmpeg** .github release tagged as **n0.9** and a fork of the **chromaprint** .github project **v1.2**, and were specifically built using the source, scripts, and makefiles available at the phorton1/chromaprint repository.

Please see the phorton1/chromaprint repository for a discussion of why this might be important.


