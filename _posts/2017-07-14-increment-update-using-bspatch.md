---
layout: post
title: 使用bspatch和bsdiff进行APK的增量更新
date: 2017-7-14
categories: blog
tags: [Android]
description: 使用bspatch和bsdiff进行APK的增量更新
---

### bsdiff简介

[bsdiff](http://www.daemonology.net/bsdiff/)和bspatch是一组开源的二进制查分工具，利用bsdiff对新旧apk文件查分生成补丁文件，再利用bspatch应用补丁文件与旧apk文件即可生成新apk文件，这样我们就可以实现android版本增量更新。
相对于传统的下载全新版本的apk，客户端只需下载服务端查分新旧版本生成的补丁文件，因此可以节约企业带宽，好一些应用商店的省流量更新功能就是这样实现的，腾讯的热修复项目[Tinker](https://github.com/Tencent/tinker)也使用了bspatch。

bsdiff和bspatch使用了[bzip2](http://www.bzip.org/downloads.html)压缩工具。

使用方法：

`bsdiff oldfile newfile patchfile`

`bspatch oldfile newfile patchfile`

效果图
![效果图](https://raw.githubusercontent.com/maoinbupt/blog.io/master/img/bspatch-ezgif.com-video-to-gif.gif)


### 创建项目

Android Studio 2.2集成了cmake构建工具，NDK开发更加方便了，我们在创建新项目只需勾选`include c++ support`即可，studio会在application根目录创建一个CMakeLists.txt用来构建c或c++代码，app的build.gradle需指定cmakelist路径及关联原生库，以及指定编译平台，代码如下：


	android{
	
		……
		defaultConfig{
			……
			externalNativeBuild {
	            cmake {
	                cppFlags "-frtti -fexceptions"
	            }
	        }
			ndk {
	            moduleName "bspatcher" 
	            abiFilters "armeabi" //编译支持的平台
	        }
		}
		
		externalNativeBuild {
	        cmake {
	            path "CMakeLists.txt"
	        }
	    }
		……
}



### 导入bsdiff和bzip2的源码

然后将下载好的bsdiff和bzip2的源码中的.c和.h文件拷贝到src/main/jni目录下，修改CMakeList.txt如下：

	add_library{

		bspatcher
			
		SHARED
		
		src/main/jni/bspatch.c
		src/main/jni/blocksort.c
	    src/main/jni/bsdiff.c
	    src/main/jni/bzip2.c
	    src/main/jni/bzip2recover.c
	    src/main/jni/bzlib.c
	    src/main/jni/bzlib.h
	    src/main/jni/bzlib_private.h
	    src/main/jni/compress.c
	    src/main/jni/crctable.c
	    src/main/jni/decompress.c
	    src/main/jni/dlltest.c
	    src/main/jni/huffman.c
	    src/main/jni/mk251.c
	    src/main/jni/randtable.c
	    src/main/jni/spewG.c
	    src/main/jni/unzcrash.c
		}
		target_link_libraries{
			bspatcher
	}


修改c代码中包含main函数的类，防止构建时出现多个入口的错误，修改bspatch.c如下，加入我们要调用的方法

	……
	int bspatch(int argc,char * argv[]){
	……
	}
	
	JNIEXPORT jint Java_cn_reamongao_bspatchdemo_BSPatch_bspatcher(JNIEnv *env,  jclass type, jstring oldfile, jstring newfile, jstring patchfile) {
	
	const char *oldApkPath = (*env)->GetStringUTFChars(env, oldfile, 0);
	const char *newApkPath = (*env)->GetStringUTFChars(env, newfile, 0);
	const char *patchPath = (*env)->GetStringUTFChars(env, patchfile, 0);
	
	int argc = 4;
	char* argv[4];
	argv[0] = "bspatch";
	argv[1] = oldApkPath;
	argv[2] = newApkPath;
	argv[3] = patchPath;
	
	int ret =  bspatch(argc, argv);
	(*env)->ReleaseStringUTFChars(env, oldfile, oldApkPath);
	(*env)->ReleaseStringUTFChars(env, newfile, newApkPath);
	(*env)->ReleaseStringUTFChars(env, patchfile, patchPath);
	
	return ret;
	
}

看源码可以知道bspatch方法第一个参数应该是4，第二个参数是字符数组，第一个随意，二三四分别为旧文件，新文件和patch文件的路径。


### 创建调用native方法的java类

	public class BSPatch {
	
	    public static native int bspatcher(String oldfile, String newfile, String patchfile);
	
	    static {
	        System.loadLibrary("bspatcher");
	    }
}

### 请开始你的表演

下面是MainActivity的代码，在onCreate里异步下载patch文件，点击button开始增量更新，

	package cn.reamongao.bspatchdemo
	
	import android.content.Context
	import android.content.Intent
	import android.content.pm.PackageManager
	import android.net.Uri
	import android.os.AsyncTask
	import android.os.Bundle
	import android.os.Environment
	import android.os.Message
	import android.support.v7.app.AppCompatActivity
	import android.util.Log
	import android.widget.Button
	import android.widget.Toast
	import org.jetbrains.anko.toast
	import java.io.ByteArrayOutputStream
	import java.io.File
	import java.io.FileOutputStream
	import java.io.InputStream
	import java.net.HttpURLConnection
	import java.net.URL
	import android.support.v4.content.FileProvider
	import android.os.Build
	
	class MainActivity : AppCompatActivity() {
	
	    internal var packageName = "cn.reamongao.bspatchdemo"
	    internal var PATCH_DOMAIN = "http://10.2.148.52:8080/files/apk.patch"
	    internal var PATCH_NAME = "apk.patch"
	    internal var NEW_FILE_NAME = "app.apk"
	    internal var TAG = "MainActivity"
	    internal var MSG_TOAST = 100
	    val SDCARD_PATH = Environment.getExternalStorageDirectory().toString()
	
	    override fun onCreate(savedInstanceState: Bundle?) {
	        super.onCreate(savedInstanceState)
	        setContentView(R.layout.activity_main)
	
	        // Example of a call to a native method
	        val tv = findViewById(R.id.sample_text) as Button
	
	        tv.setOnClickListener {
	            //并行任务
	            ApkUpdateTask().executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR)
	        }
	        tv.setText("当前版本： " +getVersionCode())
	
	        if (getVersionCode()< 2){
	
	                // 1异步下载
	                try {
	                    Log.d(TAG,"开始异步下载文件...")
	                    Thread(Runnable { downLoadFromUrl(PATCH_DOMAIN, PATCH_NAME,SDCARD_PATH)}).start()
	                    Log.d(TAG,"异步下载文件完成...")
	                } catch (e: Exception) {
	                    e.printStackTrace()
	                }
	                
	            }
	
	    }
	
	    private fun getVersionCode(): Int {
	        try {
	            val packageInfo = packageManager.getPackageInfo(packageName, 0)
	            return packageInfo.versionCode
	        } catch (e: PackageManager.NameNotFoundException) {
	            e.printStackTrace()
	        }
	
	        return 0
	    }
	
	    fun installApk(path: String, context: Context) {
	        val file = File(path)
	        if (file != null && file.exists()) {
	            val intent = Intent("android.intent.action.VIEW")
	
	            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
	                intent.flags = Intent.FLAG_GRANT_READ_URI_PERMISSION
	                val contentUri = FileProvider.getUriForFile(context, BuildConfig.APPLICATION_ID + ".fileProvider", file)
	                intent.setDataAndType(contentUri, "application/vnd.android.package-archive")
	            } else {
	                intent.setDataAndType(Uri.fromFile(file), "application/vnd.android.package-archive")
	                intent.flags = Intent.FLAG_ACTIVITY_NEW_TASK
	            }
	            context.startActivity(intent)
	        }
	    }
	
	
	    /**
	     * 合并增量文件任务
	     */
	    private inner class ApkUpdateTask : AsyncTask<Void, Void, Boolean>() {
	
	        override fun doInBackground(vararg params: Void): Boolean? {
	            val oldApkPath = getCurApkPath(this@MainActivity)
	            val oldApkFile = File(oldApkPath)
	            val patchFile = File(getPatchFilePath())
	            if (oldApkFile.exists() && patchFile.exists()) {
	                Log.d(TAG,"正在合并增量文件...")
	                val newApkPath = getNewApkFilePath()
	                BSPatch.bspatcher(oldApkPath, newApkPath, getPatchFilePath())
	                return true
	            }else{
	                handler.sendMessage(handler.obtainMessage(MSG_TOAST,"patch文件不存在"))
	            }
	            return false
	        }
	
	        override fun onPostExecute(result: Boolean?) {
	            super.onPostExecute(result)
	            if (result!!) {
	                Log.d(TAG,"合并成功，开始安装")
	                installApk(getNewApkFilePath(), this@MainActivity)
	            } else {
	                Log.d(TAG,"合并失败")
	            }
	        }
	    }
	
	    private val handler = object : android.os.Handler() {
	        override fun handleMessage(msg: Message) {
	            super.handleMessage(msg)
	            if (msg.what == MSG_TOAST) {
	                toast((msg.obj) as String)
	            }
	        }
	    }
	
	    /**
	     * 获取当前应用的Apk路径
	     */
	    fun getCurApkPath(context: Context): String {
	        var context = context
	        context = context.applicationContext
	        val applicationInfo = context.applicationInfo
	        val apkPath = applicationInfo.sourceDir
	        return apkPath
	    }
	
	    /**
	     * 获取patch文件路径
	     */
	    private fun getPatchFilePath(): String {
	        return SDCARD_PATH + File.separator + PATCH_NAME
	    }
	
	    /**
	     * 获取增量更新后的Apk路径
	     */
	    private fun getNewApkFilePath(): String {
	        return SDCARD_PATH + File.separator + NEW_FILE_NAME
	    }
	
	
	    /**
	     * 从网络Url中下载文件
	     * *
	     */
	    fun downLoadFromUrl(urlStr: String, fileName: String, savePath: String) {
	        try {
	            val url = URL(urlStr)
	            val conn = url.openConnection() as HttpURLConnection
	            //设置超时间为3秒
	            conn.setConnectTimeout(3 * 1000)
	            //防止屏蔽程序抓取而返回403错误
	            conn.setRequestProperty("User-Agent", "Mozilla/4.0 (compatible; MSIE 5.0; Windows NT; DigExt)")
	
	            //得到输入流
	            val inputStream = conn.getInputStream()
	            //获取自己数组
	            val getData = readInputStream(inputStream)
	
	            //文件保存位置
	            val saveDir = File(savePath)
	            if (!saveDir.exists()) {
	                saveDir.mkdir()
	            }
	            var path = ""
	            if (savePath.endsWith(File.separator)) {
	                path = saveDir.toString() + fileName
	            } else {
	                path = saveDir.toString() + File.separator + fileName
	            }
	            val file = File(path)
	            val fos = FileOutputStream(file)
	            fos.write(getData)
	            fos?.close()
	            if (inputStream != null) {
	                inputStream!!.close()
	            }
	
	            handler.sendMessage(handler.obtainMessage(MSG_TOAST, "patch文件下载完成"))
	
	            println("info:$url download success")
	        } catch (e: Exception) {
	            e.printStackTrace()
	        }
	    }
	
	    /**
	     * 从输入流中获取字节数组
	     * @param inputStream
	     */
	    public fun readInputStream(stream: InputStream): ByteArray? /* nullable */ {
	        val out = ByteArrayOutputStream(8192) /* `out` is a keyword, it may be better to choose another name for var */
	        val buffer = ByteArray(8192)
	
	        try {
	            while (true) {
	                val length = stream.read(buffer)
	                if (length <= 0)
	                    break
	                out.write(buffer, 0, length)
	            }
	            out.flush()
	            return out.toByteArray()
	        } catch (t: Throwable /* it's better to catch specific exception here, or, at least, Exception, but not Throwable */) {
	            // behave
	        }
	        return null
	    }
	
}

### 源码

源码请移步[GitHub](https://github.com/maoinbupt/workspace/tree/master/BSPatchDemo)
















