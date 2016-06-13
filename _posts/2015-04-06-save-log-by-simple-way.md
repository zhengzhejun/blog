---
layout: post
title: Log保存文件-Android
comments: true
category: Android
tags: [log, logcat, LogService]
---

## 前言

在开发过程中尤其是项目进入测试后，为了方便定位问题，经常需要将log保存下来，当然你可以自己写个log辅助类又或者直接使用第三方的jar包，这里多多少少需要对系统的log类进行一些包装，调用也不再是系统的log类，如果你只希望调用系统的log类又想打印的log能按自己的规则保存到文件那么请继续往下看。

## 原理

首先启动模拟器或者用手机连接电脑，进入命令行并且输入如下命令

	C:\Users\Administrator>adb shell
	root@pisces:/ # logcat -v time *:W
	05-01 17:10:48.619 W/PushService( 2216): 2015-05-01 17:10:48,620 - [WARN::PushService] - [Process:2216][Thread:1] Service called on timer
	05-01 17:10:48.622 W/PushService( 2216): 2015-05-01 17:10:48,623 - [WARN::PushService] - [Process:2216][Thread:31] SMACK 05:10:48 下午 SENT (1122316296): <iq to='xiaomi.com' id='0' chid='0' type='get'><ping xmlns='urn:xmpp:ping'><pf><p>t:69</p></pf></ping></iq>
	05-01 17:10:48.622 W/PushService( 2216): 
	05-01 17:10:48.701 W/PushService( 2216): 2015-05-01 17:10:48,701 - [WARN::PushService] - [Process:2216][Thread:428] SMACK 05:10:48 下午 RCV  (1122316296): <iq chid='0' id='0' type='result'/>
	05-01 17:11:03.638 W/PushService( 2216): 2015-05-01 17:11:03,637 - [WARN::PushService] - [Process:2216][Thread:31] JOB: check the ping-pong.
	05-01 17:11:08.384 W/MountService(  960): getVolumeState(/storage/emulated/0/external_sd): Unknown volume
	05-01 17:11:08.385 W/MountService(  960): getVolumeState(/mnt/sdcard-ext): Unknown volume
	05-01 17:11:08.386 W/MountService(  960): getVolumeState(/mnt/ext_sdcard): Unknown volume
	05-01 17:11:08.387 W/MountService(  960): getVolumeState(/mnt/sdcard2): Unknown volume
	
上面的命令过滤出了所有的w以及w优先级以上的log（本人D等级打印太多所以选用W），按“Ctrl + c”终止刚才的命令，然后再执行如下命令

	C:\Users\Administrator>adb shell
	root@pisces:/ # logcat -v time *:W >> /sdcard/log.log
	logcat -v time *:W > sdcard/log.log

这时候你再去设备的sd卡根目录看看，是不是多出来个“log.log”文件，打开看看里面的内容是不是我们保存的log。

是的，这里我们要用到的就是上面的方式，既然命令行中的log可以保存的文件中，那我们只需要在程序中执行命令行即可

## logcat

关于logcat的用法可以通过如下命令查看

	C:\Users\Administrator>adb shell
	root@pisces:/ # logcat --help
	logcat --help
	Usage: logcat [options] [filterspecs]
	options include:
	  -s              Set default filter to silent.
					  Like specifying filterspec '*:s'
	  -f <filename>   Log to file. Default to stdout
	  -r [<kbytes>]   Rotate log every kbytes. (16 if unspecified). Requires -f
	  -n <count>      Sets max number of rotated logs to <count>, default 4
	  -v <format>     Sets the log print format, where <format> is one of:

					  brief process tag thread raw time threadtime long

	  -c              clear (flush) the entire log and exit
	  -d              dump the log and then exit (don't block)
	  -t <count>      print only the most recent <count> lines (implies -d)
	  -g              get the size of the log's ring buffer and exit
	  -b <buffer>     Request alternate ring buffer, 'main', 'system', 'radio'
					  or 'events'. Multiple -b parameters are allowed and the
					  results are interleaved. The default is -b main -b system.
	  -B              output the log in binary
	filterspecs are a series of
	  <tag>[:priority]

	where <tag> is a log component tag (or * for all) and priority is:
	  V    Verbose
	  D    Debug
	  I    Info
	  W    Warn
	  E    Error
	  F    Fatal
	  S    Silent (supress all output)

	'*' means '*:d' and <tag> by itself means <tag>:v

	If not specified on the commandline, filterspec is set from ANDROID_LOG_TAGS.
	If no filterspec is found, filter defaults to '*:I'

	If not specified with -v, format is set from ANDROID_PRINTF_LOG
	or defaults to "brief"
	
从logcat help可以看出，logcat本身就提供了保存文件的参数“-f <filename>”，你可能要问为什么不用直接这个保存，主要是logcat也有它的局限性，从上面可以看出，logcat提供了根据tag和priority的过滤器，但如果我们想把项目中的所有log都保存到文件中，那我们就不得不将所有的tag都添加到过滤器中，这时候如果可以根据应用程序的包名或者进程ID过滤就省事多了，所以我们有用到了grep命令。

## grep

grep命令的用法如下

	root@pisces:/ # grep
	grep
	usage: grep [-abcDEFGHhIiJLlmnOoPqRSsUVvwxZz] [-A num] [-B num] [-C[num]]
			[-e pattern] [-f file] [--binary-files=value] [--color=when]
			[-e pattern] [-f file] [--binary-files=value] [--color=when]
			[--context[=num]] [--directories=action] [--label] [--line-buffered]
			[pattern] [file ...]
	
grep命令不仅支持关键字过滤还支持正则，这里我们要用到的就是他的正则过滤。

## 使用方法

再试试如下命令
	
	logcat -v time *:W | grep '(  960):' >> /sdcard/log.txt
	
> 注： 关键字"'(  960):'"中的进程ID请修改为你系统中存在的ID号，最好直接从“logcat -v time *:W”命令输出的log中复制粘贴，另外“()”中有五个字符，进程ID长度不够在前面补空格
	
对比下SD中的log.log和log.txt你会发现log.log中包含了所有进程warn和warn以上优先级的log而log.txt中只包含了指定关键字的log。所以我们只需根据“logcat -v time *:W”打印的log的规则再配合grep命令的正则表达式是可以根据进程ID过滤的。遗憾的是本人尝试了很多次始终由于grep无法完全支持正则表达式而未能得到一个完全匹配的正则表达式，目前只得到以下表达式，虽然不能做到百分百的过滤但也算比较靠谱，如果你写出来了麻烦也告诉我一下<yxmsw2007@gmail.com>

	logcat -v time *:W | grep '^.*\/.*(  960):' >> /sdcard/log.log

## LogService

```java

package com.example.service;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Calendar;
import java.util.Comparator;
import java.util.Date;
import java.util.List;

import android.app.AlarmManager;
import android.app.PendingIntent;
import android.app.Service;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.os.Environment;
import android.os.IBinder;
import android.os.PowerManager;
import android.os.StatFs;
import android.os.PowerManager.WakeLock;
import android.os.StrictMode;
import android.util.Log;

/**
 * @project SampleAndroid   
 * @class LogService 
 * @description 
 * 日志服务，日志默认会存储在内存中的安装目录下面并且系统每隔一段时间就会检测一下log文件的大小，
 * 如果log文件超过指定的大小则会执行如下动作：
 * 1.SD卡不可用，所有log文件都保存在内存中，当log文件数超出指定的数量就会删除较早的log
 * 2.SD卡可用而且存储空间足够则将文件移动SD卡中，并检测SD中保存的文件是否都在指定时间日期内，如果不是则删除
 * 3.SD卡可用但存储空间小于指定的大小，在操作log前会先删除SD卡中的所有log并重新检测空间，如果空间仍不足则按1操作否则按2进行
 * @author yxmsw2007
 * @version 1.0
 * @email yxmsw2007@gmail.com  
 * @data 2015-5-1 下午8:18:27    
 */
public class LogService extends Service {

	private static final String TAG = LogService.class.getSimpleName();

	private static final int MEMORY_LOG_FILE_MAX_SIZE = 10 * 1024 * 1024; // 内存中日志文件最大值，10M
	private static final int MEMORY_LOG_FILE_MONITOR_INTERVAL = 10 * 60 * 1000; // 内存中的日志文件大小监控时间间隔，10分钟
	private static final int MEMORY_LOG_FILE_MAX_NUMBER = 2; // 内存中允许保存的最大文件个数
	private static final int SDCARD_LOG_FILE_SAVE_DAYS = 7; // sd卡中日志文件的最多保存天数
	private static final String LOG_FOLDER = "Log";	//日志文件夹
	private static final String MONITOR_LOG_PRIORITY = "D"; // 监听日志的最低优先级
	
	private static final String MONITOR_LOG_SIZE_ACTION = "MONITOR_LOG_SIZE"; // 日志文件大小监测action

	/**
	 * logcat -v time *:D | grep -n '^.*\/.*(  828):' >> /sdcard/download/Log.log
	 */
	private static String LOG_FILTER_COMMAND_FORMAT = "logcat -v time *:%s | grep '^.*\\/.*(%s):' >> %s";
	
	private SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HHmmss");// 日志名称格式

	private String mLogFileDirMemory; // 日志文件在内存中的路径(日志文件在安装目录中的路径)
	private String mLogFileDirSdcard; // 日志文件在sdcard中的路径

	private String mCurrLogFileName; // 如果当前的日志写在内存中，记录当前的日志文件名称

	private Process process;

	private WakeLock wakeLock;

	private PendingIntent mMemoryLogFileSizeMonitor; 
	private AlarmManager mAlarmManager;

	@Override
	public IBinder onBind(Intent intent) {
		return null;
	}

	@Override
	public void onCreate() {
		super.onCreate();
		Log.d(TAG, "-- onCreate() --");
		if (android.os.Build.VERSION.SDK_INT > 9) {
		    StrictMode.ThreadPolicy policy = new StrictMode.ThreadPolicy.Builder().permitAll().build();
		    StrictMode.setThreadPolicy(policy);
		}
//		注册SD卡状态监听广播
		IntentFilter sdCarMonitorFilter = new IntentFilter();
		sdCarMonitorFilter.addAction(Intent.ACTION_MEDIA_MOUNTED);
		sdCarMonitorFilter.addAction(Intent.ACTION_MEDIA_UNMOUNTED);
		sdCarMonitorFilter.addDataScheme("file");
		registerReceiver(mSDStateReceiver, sdCarMonitorFilter);
		

		IntentFilter logTaskFilter = new IntentFilter();
		logTaskFilter.addAction(MONITOR_LOG_SIZE_ACTION); //注册日志大小监听广播
		logTaskFilter.addAction(Intent.ACTION_TIME_CHANGED); //如果时间被重新设置，要判断当前时间是否在有效时间内，
															 //如果超出了范围需要将日志保存到新文件中，否则会被清除
		registerReceiver(mLogTaskReceiver, logTaskFilter);
		
//		部署日志大小监控任务
		Intent intent = new Intent(MONITOR_LOG_SIZE_ACTION);
		mMemoryLogFileSizeMonitor = PendingIntent.getBroadcast(this, 0, intent, 0);
		mAlarmManager = (AlarmManager) getSystemService(ALARM_SERVICE);
		mAlarmManager.setRepeating(AlarmManager.RTC_WAKEUP, System.currentTimeMillis(),
				MEMORY_LOG_FILE_MONITOR_INTERVAL, mMemoryLogFileSizeMonitor);

		PowerManager pm = (PowerManager) getApplicationContext().getSystemService(Context.POWER_SERVICE);
		wakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, TAG);
		
		new LogCollectorThread().start();
	}

	@Override
	public void onDestroy() {
		super.onDestroy();
		Log.d(TAG, "-- onDestroy() --");
		if (process != null) {
			process.destroy();
		}
		
//		取消日志大小监控任务
		mAlarmManager.cancel(mMemoryLogFileSizeMonitor);
		
		unregisterReceiver(mSDStateReceiver);
		unregisterReceiver(mLogTaskReceiver);
	}

	/**
	 * 日志收集 
	 * 1.清除日志缓存 
	 * 2.杀死应用程序已开启的Logcat进程防止多个进程写入一个日志文件 
	 * 3.开启日志收集进程 
	 * 4.处理日志文件 移动 OR 删除
	 */
	class LogCollectorThread extends Thread {

		public LogCollectorThread() {
			super("LogCollectorThread");
			Log.d(TAG, "LogCollectorThread is create");
		}

		@Override
		public void run() {
			try {
				wakeLock.acquire(); // 唤醒手机
				clearLogCache();
				List<String> orgProcessList = getAllProcess();
				List<ProcessInfo> processInfoList = getProcessInfoList(orgProcessList);
				killLogcatProc(processInfoList);
				mCurrLogFileName = dateToFileName(new Date());//获取新的日志文件名称
				createLogCollector(getMemoryFilePath(mCurrLogFileName));
				Thread.sleep(1000);// 休眠，创建文件，然后处理文件，不然该文件还没创建，会影响文件删除
				handleLog(mCurrLogFileName);
				wakeLock.release(); // 释放
			} catch (Exception e) {
				e.printStackTrace();
			}
		}
	}

	/**
	 * 每次记录日志之前先清除日志的缓存, 不然会在两个日志文件中记录重复的日志
	 */
	private void clearLogCache() {
		Log.d(TAG, "-- clearLogCache() --");
		Process proc = null;
		List<String> commandList = new ArrayList<String>();
		commandList.add("logcat");
		commandList.add("-c");
		try {
			proc = Runtime.getRuntime().exec(commandList.toArray(new String[commandList.size()]));
			StreamConsumer errorGobbler = new StreamConsumer(proc.getErrorStream());
			StreamConsumer outputGobbler = new StreamConsumer(proc.getInputStream());
			errorGobbler.start();
			outputGobbler.start();
			if (proc.waitFor() != 0) {
				Log.e(TAG, " clearLogCache proc.waitFor() != 0");
			}
		} catch (Exception e) {
			Log.e(TAG, "clearLogCache failed", e);
		} finally {
			try {
				proc.destroy();
			} catch (Exception e) {
				Log.e(TAG, "clearLogCache failed", e);
			}
		}
	}

	/**
	 * 关闭由本程序开启的logcat进程： 根据用户名称杀死进程(如果是本程序进程开启的Logcat收集进程那么两者的USER一致)
	 * 如果不关闭会有多个进程读取logcat日志缓存信息写入日志文件
	 * @param allProcList
	 * @return
	 */
	private void killLogcatProc(List<ProcessInfo> allProcList) {
		Log.d(TAG, "-- killLogcatProc() --");
		if (process != null) {
			process.destroy();
		}
		String packName = this.getPackageName();
		String myUser = getAppUser(packName, allProcList);
		for (ProcessInfo processInfo : allProcList) {
			if (processInfo.name.toLowerCase().equals("logcat")
					&& processInfo.user.equals(myUser)) {
				android.os.Process.killProcess(Integer.parseInt(processInfo.pid));
			}
		}
	}

	/**
	 * 获取本程序的用户名称
	 * @param packName
	 * @param allProcList
	 * @return
	 */
	private String getAppUser(String packName, List<ProcessInfo> allProcList) {
		for (ProcessInfo processInfo : allProcList) {
			if (processInfo.name.equals(packName)) {
				return processInfo.user;
			}
		}
		return null;
	}

	/**
	 * 根据ps命令得到的内容获取PID，User，name等信息
	 * @param orgProcessList
	 * @return
	 */
	private List<ProcessInfo> getProcessInfoList(List<String> orgProcessList) {
		List<ProcessInfo> procInfoList = new ArrayList<ProcessInfo>();
		for (int i = 1; i < orgProcessList.size(); i++) {
			String processInfo = orgProcessList.get(i);
			String[] proStr = processInfo.split(" ");
			// USER PID PPID VSIZE RSS WCHAN PC NAME
			// root 1 0 416 300 c00d4b28 0000cd5c S /init
			List<String> orgInfo = new ArrayList<String>();
			for (String str : proStr) {
				if (!"".equals(str)) {
					orgInfo.add(str);
				}
			}
			if (orgInfo.size() == 9) {
				ProcessInfo pInfo = new ProcessInfo();
				pInfo.user = orgInfo.get(0);
				pInfo.pid = orgInfo.get(1);
				pInfo.ppid = orgInfo.get(2);
				pInfo.name = orgInfo.get(8);
				procInfoList.add(pInfo);
			}
		}
		return procInfoList;
	}

	/**
	 * 运行PS命令得到进程信息
	 * @return USER PID PPID VSIZE RSS WCHAN    PC         NAME 
	 * 		   root 1   0    416   300 c00d4b28 0000cd5c S /init
	 */
	private List<String> getAllProcess() {
		List<String> orgProcList = new ArrayList<String>();
		Process proc = null;
		try {
			proc = Runtime.getRuntime().exec("ps");
			StreamConsumer errorConsumer = new StreamConsumer(
					proc.getErrorStream());

			StreamConsumer outputConsumer = new StreamConsumer(
					proc.getInputStream(), orgProcList);

			errorConsumer.start();
			outputConsumer.start();
			if (proc.waitFor() != 0) {
				Log.e(TAG, "getAllProcess proc.waitFor() != 0");
			}
		} catch (Exception e) {
			Log.e(TAG, "getAllProcess failed", e);
		} finally {
			try {
				proc.destroy();
			} catch (Exception e) {
				Log.e(TAG, "getAllProcess failed", e);
			}
		}
		return orgProcList;
	}
	
	/**
	 * eg: logcat -v time *:D | grep -n '^.*\/.*(  828):' >> /sdcard/download/Log.log
	 * @param priority
	 * @param path
	 * @return
	 */
	private String getLogFilterCommand(String priority, String path) {
		Log.d(TAG, "-- getLogFilterCommand() --");
		String sid = "" + android.os.Process.myPid();
		for (int i = sid.length(); i < 5; i++) {
			sid = " " + sid;
		}
		return String.format(LOG_FILTER_COMMAND_FORMAT, priority, sid, path);
	}

	/**
	 * 开始收集日志信息
	 */
	public void createLogCollector(String path) {
		Log.d(TAG, "-- createLogCollector() --");
		String command = getLogFilterCommand(MONITOR_LOG_PRIORITY, path.replace(" ", "\\ "));//获取命令
		Log.d(TAG, command);
		List<String> commandList = new ArrayList<String>();
		commandList.add("sh");
		commandList.add("-c");
		commandList.add(command);
		try {
			process = Runtime.getRuntime().exec(commandList.toArray(new String[commandList.size()]));
			// process.waitFor();
		} catch (Exception e) {
			Log.e(TAG, "CollectorThread == >" + e.getMessage(), e);
		}
	}

	/**
	 * 处理日志文件 
	 */
	public void handleLog(String currFileName) {
		Log.d(TAG, "-- handleLog() --");
		moveLogfile(currFileName);//将内存中日志文件转移到SD中，当前正在记录的日志文件除外
		deleteSdcardExpiredLog(SDCARD_LOG_FILE_SAVE_DAYS);//删除SD中过期的日志文件
		deleteMemoryExpiredLog(MEMORY_LOG_FILE_MAX_NUMBER, currFileName);//删除内存中过期的日志文件
	}

	/**
	 * 检查日志文件大小是否超过了规定大小 如果超过了重新开启一个日志收集进程
	 */
	private void checkLogSize() {
		Log.d(TAG, "-- checkLogSize() --");
		if (mCurrLogFileName != null && !"".equals(mCurrLogFileName)) {
			String path = getMemoryFilePath(mCurrLogFileName);
			File file = new File(path);
			if (!file.exists()) {
				return;
			}
			if (file.length() >= MEMORY_LOG_FILE_MAX_SIZE) {
				Log.d(TAG, "The log's size is too big!");
				new LogCollectorThread().start();
			}
		}
	}
	
	/**
	 * 将日志文件转移到SD卡下面
	 */
	private void moveLogfile(String currFileName) {
		Log.d(TAG, "-- moveLogfile() --");
		if (!isSdcardAvailable()) {
			return;
		}
		File file = new File(getMemoryDirPath());
		if (!file.isDirectory()) {
			return;
		}
		File[] allFiles = file.listFiles();
		for (File logFile : allFiles) {
			String fileName = logFile.getName();
			if (currFileName.equals(fileName)) {
				continue;
			}
			if (logFile.length() >= getSdcardAvailableSize()) {
				deleteSdcardAllLog();
			}
			if (logFile.length() >= getSdcardAvailableSize()) {
				return;
			}
			if (copy(logFile, new File(getSdcardFilePath(fileName)))) {
				logFile.delete();
				Log.d(TAG, "move file success, log name is:" + fileName);
			}
		}
	}

	/**
	 * 删除SD卡中所有日志文件
	 */
	private void deleteSdcardAllLog() {
		Log.d(TAG, "-- deleteSdcardAllLog() --");
		if (isSdcardAvailable()) {
			File file = new File(getSdcardDirPath());
			if (file.isDirectory()) {
				File[] allFiles = file.listFiles();
				for (File logFile : allFiles) {
					logFile.delete();
					Log.d(TAG, "delete file success, file name is:" + logFile.getName());
				}
			}
		}
	}

	/**
	 * 删除SD内过期的日志
	 */
	private void deleteSdcardExpiredLog(int days) {
		Log.d(TAG, "-- deleteSdcardExpiredLog() --");
		if (isSdcardAvailable()) {
			File file = new File(getSdcardDirPath());
			File[] allFiles = file.listFiles();
			for (File logFile : allFiles) {
				if (canDeleteSDLog(logFile.getName(), days)) {
					logFile.delete();
					Log.d(TAG, "delete file success, file name is:" + logFile.getName());
				}
			}
		}
	}

	/**
	 * 
	 * @param createDateStr
	 * @return
	 */
	/**
	 * 判断sdcard上的日志文件是否可以删除，规则：文件创建时间是否在days天数内
	 * @param fileName 文件名
	 * @param days 天数
	 * @return
	 */
	public boolean canDeleteSDLog(String fileName, int days) {
		Calendar calendar = Calendar.getInstance();
		calendar.add(Calendar.DAY_OF_MONTH, -1 * days);// 删除7天之前日志
		Date expiredDate = calendar.getTime();
		try {
			Date createDate = fileNameToDate(fileName);
			return createDate.before(expiredDate);
		} catch (ParseException e) {
			Log.e(TAG, e.getMessage(), e);
		}
		return false;
	}

	/**
	 * 删除内存中的过期日志，删除规则： 保存最近的number个日志文件
	 * @param number 保留的文件个数
	 */
	private void deleteMemoryExpiredLog(int number, String currFileName) {
		Log.d(TAG, "-- deleteMemoryExpiredLog() --");
		File file = new File(getMemoryDirPath());
		if (file.isDirectory()) {
			File[] allFiles = file.listFiles();
			Arrays.sort(allFiles, new FileComparator());
			for (int i = 0; i < allFiles.length - number; i++) {
				File f = allFiles[i];
				if (f.getName().equals(currFileName)) {
					continue;
				}
				f.delete();
				Log.d(TAG, "delete file success, file name is:" + f.getName());
			}
		}
	}

	/**
	 * 拷贝文件
	 * @param source
	 * @param target
	 * @return
	 */
	private boolean copy(File source, File target) {
		FileInputStream in = null;
		FileOutputStream out = null;
		try {
			if (!target.exists()) {
				boolean createSucc = target.createNewFile();
				if (!createSucc) {
					return false;
				}
			}
			in = new FileInputStream(source);
			out = new FileOutputStream(target);
			byte[] buffer = new byte[8 * 1024];
			int count;
			while ((count = in.read(buffer)) != -1) {
				out.write(buffer, 0, count);
			}
			return true;
		} catch (Exception e) {
			Log.e(TAG, e.getMessage(), e);
			return false;
		} finally {
			try {
				if (in != null) {
					in.close();
				}
				if (out != null) {
					out.close();
				}
			} catch (IOException e) {
				Log.e(TAG, e.getMessage(), e);
				return false;
			}
		}
	}

	class ProcessInfo {
		public String user;
		public String pid;
		public String ppid;
		public String name;

		@Override
		public String toString() {
			return "user=" + user + " pid=" + pid + " ppid=" + ppid + " name=" + name;
		}
	}

	class StreamConsumer extends Thread {
		InputStream is;
		List<String> list;

		StreamConsumer(InputStream is) {
			this.is = is;
		}

		StreamConsumer(InputStream is, List<String> list) {
			this.is = is;
			this.list = list;
		}

		public void run() {
			try {
				InputStreamReader isr = new InputStreamReader(is);
				BufferedReader br = new BufferedReader(isr);
				String line = null;
				while ((line = br.readLine()) != null) {
					if (list != null) {
						list.add(line);
					}
				}
			} catch (IOException ioe) {
				ioe.printStackTrace();
			}
		}
	}

	/**
	 * 监控SD卡状态
	 */
	private BroadcastReceiver mSDStateReceiver = new BroadcastReceiver() {
		
		public void onReceive(Context context, Intent intent) {
			String action = intent.getAction();
			Log.d(TAG, "mLogTaskReceiver： " + action);
			if (Intent.ACTION_MEDIA_UNMOUNTED.equals(intent.getAction())) { // 存储卡被卸载
				
			} else { // 存储卡被挂载
				
			}
		}
		
	};

	/**
	 * 监控日志大小
	 */
	private BroadcastReceiver mLogTaskReceiver = new BroadcastReceiver() {
		
		public void onReceive(Context context, Intent intent) {
			String action = intent.getAction();
			Log.d(TAG, "mLogTaskReceiver： " + action);
			if (MONITOR_LOG_SIZE_ACTION.equals(action)) {
				checkLogSize();
			} else if (Intent.ACTION_TIME_CHANGED.equals(action)) {
				if (canDeleteSDLog(mCurrLogFileName, SDCARD_LOG_FILE_SAVE_DAYS)) {
					Log.d(TAG, "The log is out of date !");
					new LogCollectorThread().start();
				}
			}
		}
		
	};

	private class FileComparator implements Comparator<File> {
		
		public int compare(File file1, File file2) {
			try {
				Date create1 = fileNameToDate(file1.getName());
				Date create2 = fileNameToDate(file2.getName());
				if (create1.before(create2)) {
					return -1;
				} else {
					return 1;
				}
			} catch (ParseException e) {
				return 0;
			}
		}
		
	}
	
	/**
	 * SD卡是否可用
	 * @return
	 */
	private boolean isSdcardAvailable() {
		return Environment.getExternalStorageState().equals(Environment.MEDIA_MOUNTED);
	}
	
	/**
	 * SD卡剩余空间
	 * @return
	 */
	private long getSdcardAvailableSize() {
		if (isSdcardAvailable()) {
			String path = Environment.getExternalStorageDirectory().getAbsolutePath();
			StatFs fileStats = new StatFs(path);
			fileStats.restat(path);
			return fileStats.getAvailableBlocksLong() * fileStats.getBlockSizeLong();
		}
		return 0;
	}
	
	public boolean creatDir(String path) {
		File f = new File(path);
		return f.exists() && f.isDirectory() || f.mkdirs();
	}
	
	public boolean creatFile(String path) {
		if (path != null && path.contains(File.separator) 
				&& creatDir(path.substring(0, path.lastIndexOf(File.separator)))) {
			File f = new File(path);
			try {
				return f.exists() && f.isFile() || f.createNewFile();
			} catch (IOException e) {
				Log.w(TAG, e);
			}
		}
		return false;
	}
	
	/**
	 * 日志文件名转日期
	 * @param fileName
	 * @return
	 * @throws ParseException
	 */
	private Date fileNameToDate(String fileName) throws ParseException {
		fileName = fileName.substring(0, fileName.indexOf("."));//去除文件的扩展类型（.log）
		return sdf.parse(fileName);
	}
	
	/**
	 * 日期转日志文件名
	 * @param date
	 * @return
	 */
	private String dateToFileName(Date date) {
		return sdf.format(date) + ".log";//日志文件名称
	}
	
	/**
	 * 文件在内存中的路径
	 * @param fileName
	 * @return
	 */
	private String getMemoryFilePath(String fileName) {
		return getMemoryDirPath() + File.separator + fileName;
	}
	
	/**
	 * 内存中日志文件保存目录
	 * @return
	 */
	private String getMemoryDirPath() {
		if (mLogFileDirMemory == null) {
			mLogFileDirMemory = getFilesDir().getAbsolutePath() + File.separator + LOG_FOLDER;
			creatDir(mLogFileDirMemory);
		}
		return mLogFileDirMemory;
	}
	
	/**
	 * 文件在SD中的路径
	 * @param fileName
	 * @return
	 */
	private String getSdcardFilePath(String fileName) {
		return getSdcardDirPath() + File.separator + fileName;
	}
	
	/**
	 * SD卡中日志文件保存目录
	 * @return
	 */
	private String getSdcardDirPath() {
		if (mLogFileDirSdcard == null && isSdcardAvailable()) {
			mLogFileDirSdcard = Environment.getExternalStorageDirectory().getAbsolutePath() 
					+ File.separator
					+ getString(R.string.app_name)
					+ File.separator
					+ LOG_FOLDER;
			creatDir(mLogFileDirSdcard);
		}
		return mLogFileDirSdcard;
	}
	
}

```

## 源码下载

[SampleAndroid](https://github.com/yxmsw2007/SampleAndroid.git)
	
## 参考资料
[android 写log到文件](http://lovelease.iteye.com/blog/1886907)

