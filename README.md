# native-crash
android native crash 分析方法：
准备工作，log 即crash stack。addr2line工具，对应软件版本的出错.so库文件

分析过程如下
step1，crash stack分析：
01-01 12:25:40.982  8689  8710 F libc    : Fatal signal 8 (SIGFPE), code -6, fault addr 0x21f1 in tid 8710 (Thread-455)
 : backtrace:
 :     #00 pc 00044320  /system/lib/libc.so (tgkill+12)
 :     #01 pc 00041f21  /system/lib/libc.so (pthread_kill+32)
 :     #02 pc 0001ba4f  /system/lib/libc.so (raise+10)
 :     #03 pc 0000c2d8  /system/lib/libcutils.so (__aeabi_ldiv0+8)
 :     #04 pc 000bc149  /system/lib/libstagefright.so (_ZNK7android16NuMediaExtractor17getCachedDurationEPxPb+164)    => 出错的库
:     #05 pc 000241cb  /system/lib/libmedia_jni.so
:     #06 pc 72ab69b1  /data/dalvik-cache/arm/system@framework@boot.oat (offset 0x1f2a000)

01-01 12:25:41.850  2881  9036 W ActivityManager:   Native crash
01-01 12:25:41.850  2881  9036 W ActivityManager:   Native crash: Floating point exception  =》错误信息，浮点数异常

step2， 反汇编.so库：
执行反汇编命令 addr2line -fe  libstagefright.so 000bc149得到如下结果：
_ZNK7android16NuMediaExtractor17getCachedDurationEPxPb
/local/build/PM80C-release/vRUC/frameworks/av/media/libstagefright/NuMediaExtractor.cpp:576  =》找到源码：

step3，源码分析：
554bool NuMediaExtractor::getCachedDuration(
555        int64_t *durationUs, bool *eos) const {
556    Mutex::Autolock autoLock(mLock);
。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。

576        *durationUs = cachedDataRemaining * 8000000ll / bitrate;  =》这里出错，结合错误信息，浮点数异常可以推断 除0 出错，bitrate =0，查bitrate来源
。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。
581    return false;
582}

538 bool NuMediaExtractor::getTotalBitrate(int64_t *bitrate) const {
539    if (mTotalBitrate >= 0) {
540        *bitrate = mTotalBitrate;    =》bitrate =0 来源如此，
541        return true;
542    }
。。。。。。。。。。。。。。。。。
551}

step4，源码修改：patch：把 if (mTotalBitrate >= 0) 改成 if (mTotalBitrate >0)
