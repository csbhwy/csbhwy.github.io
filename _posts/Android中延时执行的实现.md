##### 第一种

```java
new Thread(new Runnable() {
    @Override
    public void run() {
        //sleep设置的是时长
        Thread.sleep(3000);
        //TODO
        //如果是更新UI，可以延时发送异步消息到主线程通知更新
        //handler.sendMessage();
    }
}).start
```
涉及到更新UI或者可以这样：

```java
handler.sendMessageDelayed(message, 3000);
```
##### 第二种

```java
//使用延时器实现
TimerTask task = new TimerTask() {
	@Override
    public void run() {
		//TODO
    }
};
Timer timer = new Timer();
timer.schedule(task, 3000);                       
```
##### 第三种

```java
new Handler().postDelayed(new Runnable() {
    @Override
    public void run() {
        //TODO
    }
}, 3000);
```

