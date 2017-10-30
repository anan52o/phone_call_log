# phone_call_log
Call log to monitor
我们在使用Android手机打电话时，有时可能会需要对来去电通话自动录音，本文就详细讲解实现Android来去电通话自动录音的方法，大家按照文中的方法编写程序就可以完成此功能。

       来去电自动录音的关键在于如何监听手机电话状态的转变：

       1）来电的状态的转换如下（红色标记是我们要用到的状态）

       空闲（IDEL）——> 响铃（RINGING）——> 接听（ACTIVE）——> 挂断（经历DISCONNECTING——DISCONNECTED）——> 空闲（IDEL） 

       或者  空闲（IDEL）——> 响铃（RINGING）——> 拒接 ——> 空闲（IDEL）

       2）去电状态的转换如下

       空闲（IDEL）——> 拨号 （DIALING）——> （对方）响铃（ALERTING） ——> 建立连接（ACTIVE）—— 挂断（经历DISCONNECTING——DISCONNECTED）——> 空闲（IDEL） 

       或者 空闲（IDEL）——> 拨号 （DIALING）——> （对方）响铃（ALERTING）——> 挂断/对方拒接 ——> 空闲（IDEL）

       下面就分别就来电和去电这两种状态分析并实现。

       1、先进行来电的分析和实现。

       相对去电来说，来电状态的转换检测要简单些。android api 中的PhoneStateListener 类提供了相应的方法，但我们需要覆盖其中的 onCallStateChanged(int state, String incomingNumber) 方法即可实现来电状态的检测，并在此基础上添加录音功能即可。其中 state 参数就是各种电话状态，到时我们将它跟下面我们要用到的状态进行比较，若是电话处在我们想要的状态上，则进行一系列操作，否则就不管他。想要获取这些状态，还需要另一个电话相关类，那就是 TelephonyManager， 该类 提供了一些电话状态，其中我们要用到的是：TelephonyManager.CALL_STATE_IDLE（空闲）、TelephonyManager.CALL_STATE_OFFHOOK（摘机）和 TelephonyManager.CALL_STATE_RINGING（来电响铃）这三个状态。判别这三种状态，可以继承 android.telephony.PhoneStateListener 类，实现上面提到的 onCallStateChanged(int state, String incomingNumber) 方法，请看如下代码：

Java代码
public class TelListener extends PhoneStateListener {     
     
    @Override     
    public void onCallStateChanged(int state, String incomingNumber) {     
        super.onCallStateChanged(state, incomingNumber);     
     
        switch (state) {     
        case TelephonyManager.CALL_STATE_IDLE: // 空闲状态，即无来电也无去电     
            Log.i("TelephoneState", "IDLE");     
            //此处添加一系列功能代码    
            break;     
        case TelephonyManager.CALL_STATE_RINGING: // 来电响铃     
            Log.i("TelephoneState", "RINGING");     
            //此处添加一系列功能代码    
            break;     
        case TelephonyManager.CALL_STATE_OFFHOOK: // 摘机，即接通    
            Log.i("TelephoneState", "OFFHOOK");     
            //此处添加一系列功能代码    
            break;     
        }     
     
        Log.i("TelephoneState", String.valueOf(incomingNumber));     
    }     
     
}  
       有了以上来电状态监听代码还不足以实现监听功能，还需要在我们的一个Activity或者Service中实现监听，方法很简单，代码如下：

Java代码
/**   
* 在activity 或者 service中加入如下代码，以实现来电状态监听   
*/    
TelephonyManager telMgr = (TelephonyManager)context.getSystemService(    
                Context.TELEPHONY_SERVICE);    
        telMgr.listen(new TelListener(), PhoneStateListener.LISTEN_CALL_STATE);   
       这样就实现了来电状态监听功能，但要能够在设备中跑起来，这还不够，它还需要两个获取手机电话状态的权限：

XML/HTML代码
<uses-permission android:name="android.permission.READ_PHONE_STATE" />    
<uses-permission android:name="android.permission.PROCESS_OUTGOING_CALLS" />   
       这样的话就可以跑起来了。

       说到这，我想如果你可以实现录音功能的话，在此基础上实现来电自动录音就应该没什么问题了，不过请容我简单罗嗦几句。既然是来电，那么要想录音的话，那么应该就是在监听到 TelephonyManager.CALL_STATE_OFFHOOK 的状态时开启录音机开始录音， 在监听到TelephonyManager.CALL_STATE_IDLE 的状态时关闭录音机停止录音。这样，来电录音功能就完成了，不要忘记录音功能同样需要权限：

XML/HTML代码
<uses-permission android:name="android.permission.RECORD_AUDIO"/>     
     
<!-- 要存储文件或者创建文件夹的话还需要以下两个权限 -->     
<uses-permission android:name="android.permission.MOUNT_UNMOUNT_FILESYSTEMS"/>     
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>  
       2、介绍完了来电自动录音，下面就来介绍去电自动录音的实现方法。

       上面说过，相比来电状态的监听，去电的要麻烦些，甚至这种方法不是通用的，这个主要是因为android api 中没有提供去电状态监听的相应类和方法（也许我刚接触，没有找到）。刚开始网上搜索了一通也没有找到对应的解决方法，大多是 来电监听的，也就是上面的方法。不过中途发现一篇博文（后来就搜不到了），记得是查询系统日志的方式，从中找到去电过程中的各个状态的关键词。无奈之中，最终妥协了此方法。

       我的（联想A65上的）去电日志内容如下：

       过滤关键词为 mforeground

Java代码
01-06 16:29:54.225: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:54.245: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:54.631: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:54.645: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:54.742: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:54.766: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:54.873: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:54.877: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:55.108: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:55.125: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : DIALING    
01-06 16:29:57.030: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : ACTIVE    
01-06 16:29:57.155: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : ACTIVE    
01-06 16:29:57.480: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : ACTIVE    
01-06 16:29:57.598: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : ACTIVE    
01-06 16:29:59.319: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : DISCONNECTING    
01-06 16:29:59.373: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : DISCONNECTING    
01-06 16:30:00.392: D/InCallScreen(251): - onDisconnect: currentlyIdle:true ; mForegroundCall.getState():DISCONNECTED    
01-06 16:30:00.399: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): - onDisconnect: currentlyIdle:true ; mForegroundCall.getState():DISCONNECTED    
01-06 16:30:01.042: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : IDLE    
01-06 16:30:01.070: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : IDLE    
01-06 16:30:01.558: D/InCallScreen(251): onPhoneStateChanged: mForegroundCall.getState() : IDLE    
01-06 16:30:01.572: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mForegroundCall.getState() : IDLE   
       过滤关键词  mbackground

Java代码
01-06 16:29:54.226: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:54.256: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:54.638: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:54.652: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:54.743: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:54.770: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:54.875: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:54.882: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:55.109: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:55.142: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:57.031: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:57.160: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:57.481: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:57.622: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:59.319: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:29:59.373: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:30:01.042: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:30:01.070: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:30:01.559: D/InCallScreen(251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE    
01-06 16:30:01.573: V/LogInfo OutGoing Call(2492): D/InCallScreen(  251): onPhoneStateChanged: mBackgroundCall.getState() : IDLE   
       从上面的日志可以看到，每一行的末尾的大写英文词就是去电的状态，状态说明如下：

       DIALING 拨号，对方还未响铃
       ACTIVE   对方接通，通话建立
       DISCONNECTING 通话断开时
       DISCONNECTED  通话已断开，可以认为是挂机了

       由于我拨打的是10010，没有响铃过程（电脑自动接通的够快），还少了一个状态，状态是ALERTING ，这个就是对方正在响铃的状态。

       有了这几个去电状态就好办了，现在我们要做的就是读取系统日志，然后找到这些状态，提取的关键词就是上面提到的 mforeground（前台通话状态） 和 mbackground （后台通话状态）（可能不一样的设备生成的不一样，根据自己具体设备设置，这里只提取前台的），如果读取的这一行日志中 包含 mforground ，再看看是否包含上面的状态的单词。既然说的如此，那么看看读取系统日志的代码吧。

Java代码
package com.sdvdxl.phonerecorder;    
    
import java.io.BufferedReader;    
import java.io.IOException;    
import java.io.InputStream;    
import java.io.InputStreamReader;    
    
import com.sdvdxl.outgoingcall.OutgoingCallState;    
    
import android.content.Context;    
import android.content.Intent;    
import android.util.Log;    
    
/**   
 *    
 * @author sdvdxl   
 *  找到 日志中的   
 *  onPhoneStateChanged: mForegroundCall.getState() 这个是前台呼叫状态   
 *  mBackgroundCall.getState() 后台电话   
 *  若 是 DIALING 则是正在拨号，等待建立连接，但对方还没有响铃，   
 *  ALERTING 呼叫成功，即对方正在响铃，   
 *  若是 ACTIVE 则已经接通   
 *  若是 DISCONNECTED 则本号码呼叫已经挂断   
 *  若是 IDLE 则是处于 空闲状态   
 *     
 */    
public class ReadLog extends Thread {    
    private Context ctx;    
    private int logCount;    
        
    private static final String TAG = "LogInfo OutGoing Call";    
        
    /**   
     *  前后台电话   
     * @author sdvdxl   
     *     
     */    
    private static class CallViewState {    
        public static final String FORE_GROUND_CALL_STATE = "mForeground";    
    }    
        
    /**   
     * 呼叫状态   
     * @author sdvdxl   
     *   
     */    
    private static class CallState {    
        public static final String DIALING = "DIALING";    
        public static final String ALERTING = "ALERTING";    
        public static final String ACTIVE = "ACTIVE";    
        public static final String IDLE = "IDLE";    
        public static final String DISCONNECTED = "DISCONNECTED";    
    }    
        
    public ReadLog(Context ctx) {    
        this.ctx = ctx;    
    }    
        
    /**   
     * 读取Log流   
     * 取得呼出状态的log   
     * 从而得到转换状态   
     */    
    @Override    
    public void run() {    
        Log.d(TAG, "开始读取日志记录");    
            
        String[] catchParams = {"logcat", "InCallScreen *:s"};    
        String[] clearParams = {"logcat", "-c"};    
            
        try {    
            Process process=Runtime.getRuntime().exec(catchParams);    
            InputStream is = process.getInputStream();    
            BufferedReader reader = new BufferedReader(new InputStreamReader(is));    
                
            String line = null;    
            while ((line=reader.readLine())!=null) {    
                logCount++;    
                //输出所有    
            Log.v(TAG, line);    
                    
                //日志超过512条就清理    
                if (logCount>512) {    
                    //清理日志    
                    Runtime.getRuntime().exec(clearParams)    
                        .destroy();//销毁进程，释放资源    
                    logCount = 0;    
                    Log.v(TAG, "-----------清理日志---------------");    
                }       
                    
                /*---------------------------------前台呼叫-----------------------*/    
                //空闲    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.IDLE)) {    
                    Log.d(TAG, ReadLog.CallState.IDLE);    
                }    
                    
                //正在拨号，等待建立连接，即已拨号，但对方还没有响铃，    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.DIALING)) {    
                    Log.d(TAG, ReadLog.CallState.DIALING);    
                }    
                    
                //呼叫对方 正在响铃    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.ALERTING)) {    
                    Log.d(TAG, ReadLog.CallState.ALERTING);    
                }    
                    
                //已接通，通话建立    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.ACTIVE)) {    
                    Log.d(TAG, ReadLog.CallState.ACTIVE);    
                }    
                    
                //断开连接，即挂机    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.DISCONNECTED)) {    
                    Log.d(TAG, ReadLog.CallState.DISCONNECTED);    
                }    
                    
            } //END while    
                
        } catch (IOException e) {    
            e.printStackTrace();    
        } //END try-catch    
    } //END run    
} //END class ReadLog   
       以上代码中，之所以用线程，是为了防止读取日志过程中阻滞主方法的其他方法的执行，影响到程序捕捉对应的电话状态。

       好了，捕捉到了去电过程中各个状态的转变，那么，如何通知给程序呢，我采用的方法是捕获后立马给系统发送广播，然后程序进行广播接收，接收后再处理录音事件。要发送广播，就要发送一个唯一的广播，为此，建立如下类：

Java代码
package com.sdvdxl.outgoingcall;    
    
import com.sdvdxl.phonerecorder.ReadLog;    
    
import android.content.Context;    
import android.util.Log;    
    
public class OutgoingCallState {    
    Context ctx;    
    public OutgoingCallState(Context ctx) {    
        this.ctx = ctx;    
    }    
        
    /**   
     * 前台呼叫状态   
     * @author sdvdxl   
     *   
     */    
    public static final class ForeGroundCallState {    
        public static final String DIALING =     
                "com.sdvdxl.phonerecorder.FORE_GROUND_DIALING";    
        public static final String ALERTING =     
                "com.sdvdxl.phonerecorder.FORE_GROUND_ALERTING";    
        public static final String ACTIVE =     
                "com.sdvdxl.phonerecorder.FORE_GROUND_ACTIVE";    
        public static final String IDLE =     
                "com.sdvdxl.phonerecorder.FORE_GROUND_IDLE";    
        public static final String DISCONNECTED =     
                "com.sdvdxl.phonerecorder.FORE_GROUND_DISCONNECTED";    
    }    
        
    /**   
     * 开始监听呼出状态的转变，   
     * 并在对应状态发送广播   
     */    
    public void startListen() {    
        new ReadLog(ctx).start();    
        Log.d("Recorder", "开始监听呼出状态的转变，并在对应状态发送广播");    
    }    
        
}   
       程序需要读取系统日志权限：

XML/HTML代码
<uses-permission android:name="android.permission.READ_LOGS"/>   
       然后，在读取日志的类中检测到去电各个状态的地方发送一个广播，那么，读取日志的完整代码如下：

Java代码
package com.sdvdxl.phonerecorder;    
    
import java.io.BufferedReader;    
import java.io.IOException;    
import java.io.InputStream;    
import java.io.InputStreamReader;    
    
import com.sdvdxl.outgoingcall.OutgoingCallState;    
    
import android.content.Context;    
import android.content.Intent;    
import android.util.Log;    
    
/**   
 *    
 * @author mrloong   
 *  找到 日志中的   
 *  onPhoneStateChanged: mForegroundCall.getState() 这个是前台呼叫状态   
 *  mBackgroundCall.getState() 后台电话   
 *  若 是 DIALING 则是正在拨号，等待建立连接，但对方还没有响铃，   
 *  ALERTING 呼叫成功，即对方正在响铃，   
 *  若是 ACTIVE 则已经接通   
 *  若是 DISCONNECTED 则本号码呼叫已经挂断   
 *  若是 IDLE 则是处于 空闲状态   
 *     
 */    
public class ReadLog extends Thread {    
    private Context ctx;    
    private int logCount;    
        
    private static final String TAG = "LogInfo OutGoing Call";    
        
    /**   
     *  前后台电话   
     * @author sdvdxl   
     *     
     */    
    private static class CallViewState {    
        public static final String FORE_GROUND_CALL_STATE = "mForeground";    
    }    
        
    /**   
     * 呼叫状态   
     * @author sdvdxl   
     *   
     */    
    private static class CallState {    
        public static final String DIALING = "DIALING";    
        public static final String ALERTING = "ALERTING";    
        public static final String ACTIVE = "ACTIVE";    
        public static final String IDLE = "IDLE";    
        public static final String DISCONNECTED = "DISCONNECTED";    
    }    
        
    public ReadLog(Context ctx) {    
        this.ctx = ctx;    
    }    
        
    /**   
     * 读取Log流   
     * 取得呼出状态的log   
     * 从而得到转换状态   
     */    
    @Override    
    public void run() {    
        Log.d(TAG, "开始读取日志记录");    
            
        String[] catchParams = {"logcat", "InCallScreen *:s"};    
        String[] clearParams = {"logcat", "-c"};    
            
        try {    
            Process process=Runtime.getRuntime().exec(catchParams);    
            InputStream is = process.getInputStream();    
            BufferedReader reader = new BufferedReader(new InputStreamReader(is));    
                
            String line = null;    
            while ((line=reader.readLine())!=null) {    
                logCount++;    
                //输出所有    
            Log.v(TAG, line);    
                    
                //日志超过512条就清理    
                if (logCount>512) {    
                    //清理日志    
                    Runtime.getRuntime().exec(clearParams)    
                        .destroy();//销毁进程，释放资源    
                    logCount = 0;    
                    Log.v(TAG, "-----------清理日志---------------");    
                }       
                    
                /*---------------------------------前台呼叫-----------------------*/    
                //空闲    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.IDLE)) {    
                    Log.d(TAG, ReadLog.CallState.IDLE);    
                }    
                    
                //正在拨号，等待建立连接，即已拨号，但对方还没有响铃，    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.DIALING)) {    
                    //发送广播    
                    Intent dialingIntent = new Intent();    
                    dialingIntent.setAction(OutgoingCallState.ForeGroundCallState.DIALING);    
                    ctx.sendBroadcast(dialingIntent);    
                        
                    Log.d(TAG, ReadLog.CallState.DIALING);    
                }    
                    
                //呼叫对方 正在响铃    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.ALERTING)) {    
                    //发送广播    
                    Intent dialingIntent = new Intent();    
                    dialingIntent.setAction(OutgoingCallState.ForeGroundCallState.ALERTING);    
                    ctx.sendBroadcast(dialingIntent);    
                        
                    Log.d(TAG, ReadLog.CallState.ALERTING);    
                }    
                    
                //已接通，通话建立    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.ACTIVE)) {    
                    //发送广播    
                    Intent dialingIntent = new Intent();    
                    dialingIntent.setAction(OutgoingCallState.ForeGroundCallState.ACTIVE);    
                    ctx.sendBroadcast(dialingIntent);    
                        
                    Log.d(TAG, ReadLog.CallState.ACTIVE);    
                }    
                    
                //断开连接，即挂机    
                if (line.contains(ReadLog.CallViewState.FORE_GROUND_CALL_STATE)    
                        && line.contains(ReadLog.CallState.DISCONNECTED)) {    
                    //发送广播    
                    Intent dialingIntent = new Intent();    
                    dialingIntent.setAction(OutgoingCallState.ForeGroundCallState.DISCONNECTED);    
                    ctx.sendBroadcast(dialingIntent);    
                        
                    Log.d(TAG, ReadLog.CallState.DISCONNECTED);    
                }    
                    
            } //END while    
                
        } catch (IOException e) {    
            e.printStackTrace();    
        } //END try-catch    
    } //END run    
} //END class ReadLog   
       发送了广播，那么就要有接收者，定义接收者如下（关于录音机的代码可以先忽略）：

Java代码
package com.sdvdxl.phonerecorder;    
    
import android.content.BroadcastReceiver;    
import android.content.Context;    
import android.content.Intent;    
import android.util.Log;    
    
import com.sdvdxl.outgoingcall.OutgoingCallState;    
    
public class OutgoingCallReciver extends BroadcastReceiver {    
    static final String TAG = "Recorder";    
    private MyRecorder recorder;    
        
    public OutgoingCallReciver() {    
        recorder = new MyRecorder();    
    }    
        
    public  OutgoingCallReciver (MyRecorder recorder) {    
        this.recorder = recorder;    
    }    
        
    @Override    
    public void onReceive(Context ctx, Intent intent) {    
        String phoneState = intent.getAction();    
            
        if (phoneState.equals(Intent.ACTION_NEW_OUTGOING_CALL)) {    
            String phoneNum = intent.getStringExtra(Intent.EXTRA_PHONE_NUMBER);//拨出号码    
            recorder.setPhoneNumber(phoneNum);    
            recorder.setIsCommingNumber(false);    
            Log.d(TAG, "设置为去电状态");    
            Log.d(TAG, "去电状态 呼叫：" + phoneNum);    
        }    
            
        if (phoneState.equals(OutgoingCallState.ForeGroundCallState.DIALING)) {    
            Log.d(TAG, "正在拨号...");    
        }    
            
        if (phoneState.equals(OutgoingCallState.ForeGroundCallState.ALERTING)) {    
            Log.d(TAG, "正在呼叫...");    
        }    
            
        if (phoneState.equals(OutgoingCallState.ForeGroundCallState.ACTIVE)) {    
            if (!recorder.isCommingNumber() && !recorder.isStarted()) {    
                Log.d(TAG, "去电已接通 启动录音机");    
                recorder.start();    
                    
            }    
        }    
            
        if (phoneState.equals(OutgoingCallState.ForeGroundCallState.DISCONNECTED)) {    
            if (!recorder.isCommingNumber() && recorder.isStarted()) {    
                Log.d(TAG, "已挂断 关闭录音机");    
                recorder.stop();    
            }    
        }    
    }    
    
}   
       其中有这么一段代码：

Java代码
String phoneState = intent.getAction();     
             
        if (phoneState.equals(Intent.ACTION_NEW_OUTGOING_CALL)) {     
            String phoneNum = intent.getStringExtra(Intent.EXTRA_PHONE_NUMBER);//拨出号码     
            recorder.setPhoneNumber(phoneNum);     
            recorder.setIsCommingNumber(false);     
            Log.d(TAG, "设置为去电状态");     
            Log.d(TAG, "去电状态 呼叫：" + phoneNum);     
        }   
       这里是接收系统发出的广播，用于接收去电广播。这样，就获得了去电状态。

       3、有了以上主要代码，可以说，来去电监听功能算是完成了，下面创建一个service来运行监听：

Java代码
package com.sdvdxl.service;    
    
import android.app.Service;    
import android.content.Context;    
import android.content.Intent;    
import android.content.IntentFilter;    
import android.os.IBinder;    
import android.telephony.PhoneStateListener;    
import android.telephony.TelephonyManager;    
import android.util.Log;    
import android.widget.Toast;    
    
import com.sdvdxl.outgoingcall.OutgoingCallState;    
import com.sdvdxl.phonerecorder.MyRecorder;    
import com.sdvdxl.phonerecorder.OutgoingCallReciver;    
import com.sdvdxl.phonerecorder.TelListener;    
    
public class PhoneCallStateService extends Service {    
    private OutgoingCallState outgoingCallState;    
    private OutgoingCallReciver outgoingCallReciver;    
    private MyRecorder recorder;    
        
    @Override    
    public void onCreate() {    
        super.onCreate();    
            
        //------以下应放在onStartCommand中，但2.3.5以下版本不会因service重新启动而重新调用--------    
        //监听电话状态，如果是打入且接听 或者 打出 则开始自动录音    
        //通话结束，保存文件到外部存储器上    
        Log.d("Recorder", "正在监听中...");    
        recorder = new MyRecorder();    
        outgoingCallState = new OutgoingCallState(this);    
        outgoingCallReciver = new OutgoingCallReciver(recorder);    
        outgoingCallState.startListen();    
        Toast.makeText(this, "服务已启动", Toast.LENGTH_LONG).show();    
            
        //去电    
        IntentFilter outgoingCallFilter = new IntentFilter();    
        outgoingCallFilter.addAction(OutgoingCallState.ForeGroundCallState.IDLE);    
        outgoingCallFilter.addAction(OutgoingCallState.ForeGroundCallState.DIALING);    
        outgoingCallFilter.addAction(OutgoingCallState.ForeGroundCallState.ALERTING);    
        outgoingCallFilter.addAction(OutgoingCallState.ForeGroundCallState.ACTIVE);    
        outgoingCallFilter.addAction(OutgoingCallState.ForeGroundCallState.DISCONNECTED);    
            
        outgoingCallFilter.addAction("android.intent.action.PHONE_STATE");    
        outgoingCallFilter.addAction("android.intent.action.NEW_OUTGOING_CALL");    
            
        //注册接收者    
        registerReceiver(outgoingCallReciver, outgoingCallFilter);    
            
        //来电    
        TelephonyManager telmgr = (TelephonyManager)getSystemService(    
                Context.TELEPHONY_SERVICE);    
        telmgr.listen(new TelListener(recorder), PhoneStateListener.LISTEN_CALL_STATE);    
            
            
    }    
        
    @Override    
    public IBinder onBind(Intent intent) {    
        // TODO Auto-generated method stub    
        return null;    
    }    
    
    @Override    
    public void onDestroy() {    
        super.onDestroy();    
        unregisterReceiver(outgoingCallReciver);    
        Toast.makeText(    
                this, "已关闭电话监听服务", Toast.LENGTH_LONG)    
                .show();    
        Log.d("Recorder", "已关闭电话监听服务");    
    }    
    
    @Override    
    public int onStartCommand(Intent intent, int flags, int startId) {    
            
        return START_STICKY;    
    }    
    
}   
       注册以下service：

XML/HTML代码
<service android:name="com.sdvdxl.service.PhoneCallStateService" />  
       到此为止，来去电状态的监听功能算是完成了，剩下一个录音机，附上录音机代码如下：

Java代码
package com.sdvdxl.phonerecorder;    
    
import java.io.File;    
import java.io.IOException;    
import java.text.SimpleDateFormat;    
import java.util.Date;    
    
import android.media.MediaRecorder;    
import android.os.Environment;    
import android.util.Log;    
    
public class MyRecorder {    
    private String phoneNumber;    
    private MediaRecorder mrecorder;    
    private boolean started = false; //录音机是否已经启动    
    private boolean isCommingNumber = false;//是否是来电    
    private String TAG = "Recorder";    
        
        
    public MyRecorder(String phoneNumber) {    
        this.setPhoneNumber(phoneNumber);    
    }    
        
    public MyRecorder() {    
    }    
    
    public void start() {    
        started = true;    
        mrecorder = new MediaRecorder();    
            
        File recordPath = new File(    
                Environment.getExternalStorageDirectory()    
                , "/My record");     
        if (!recordPath.exists()) {    
            recordPath.mkdirs();    
            Log.d("recorder", "创建目录");    
        }    
            
        String callDir = "呼出";    
        if (isCommingNumber) {    
            callDir = "呼入";    
        }    
        String fileName = callDir + "-" + phoneNumber + "-"     
                + new SimpleDateFormat("yy-MM-dd_HH-mm-ss")    
                    .format(new Date(System.currentTimeMillis())) + ".mp3";//实际是3gp    
        File recordName = new File(recordPath, fileName);    
            
        try {    
            recordName.createNewFile();    
            Log.d("recorder", "创建文件" + recordName.getName());    
        } catch (IOException e) {    
            e.printStackTrace();    
        }    
            
        mrecorder.setAudioSource(MediaRecorder.AudioSource.DEFAULT);    
        mrecorder.setOutputFormat(MediaRecorder.OutputFormat.DEFAULT);    
        mrecorder.setAudioEncoder(MediaRecorder.AudioEncoder.DEFAULT);    
            
        mrecorder.setOutputFile(recordName.getAbsolutePath());    
            
        try {    
            mrecorder.prepare();    
        } catch (IllegalStateException e) {    
            e.printStackTrace();    
        } catch (IOException e) {    
            e.printStackTrace();    
        }    
        mrecorder.start();    
        started = true;    
        Log.d(TAG , "录音开始");    
    }    
        
    public void stop() {    
        try {    
            if (mrecorder!=null) {    
                mrecorder.stop();    
                mrecorder.release();    
                mrecorder = null;    
            }    
            started = false;    
        } catch (IllegalStateException e) {    
            e.printStackTrace();    
        }    
            
            
        Log.d(TAG , "录音结束");    
    }    
        
    public void pause() {    
            
    }    
    
    public String getPhoneNumber() {    
        return phoneNumber;    
    }    
    
    public void setPhoneNumber(String phoneNumber) {    
        this.phoneNumber = phoneNumber;    
    }    
    
    public boolean isStarted() {    
        return started;    
    }    
    
    public void setStarted(boolean hasStarted) {    
        this.started = hasStarted;    
    }    
    
    public boolean isCommingNumber() {    
        return isCommingNumber;    
    }    
    
    public void setIsCommingNumber(boolean isCommingNumber) {    
        this.isCommingNumber = isCommingNumber;    
    }    
    
}   
到此，来去电通话自动录音的所有功能就完成了，大家可以自己试着编写并实现。