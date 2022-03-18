[DESCRIPTION]
 
 Android Q版本默认设计是关闭数据开关之后，无法发送彩信。
如果想在data disabled的情况下MO MMS, 请参考以下修改
 
 
[SOLUTION]
 需要修改三处
 
vendor/mediatek/proprietary/frameworks/opt/telephony/src/java/com/mediatek/internal/telephony/dataconnection/DataConnectionExt.java

```java
public boolean isDataAllowedAsOff(String apnType) {
	if (TextUtils.equals(apnType, PhoneConstants.APN_TYPE_DEFAULT) ||
		TextUtils.equals(apnType, PhoneConstants.APN_TYPE_MMS) || // 去掉MMS这一行
		TextUtils.equals(apnType, PhoneConstants.APN_TYPE_DUN) ||
		TextUtils.equals(apnType, MtkPhoneConstants.APN_TYPE_PREEMPT)) {
		return false;
	}
	return true;
}
```


```java
public boolean isMeteredApnType(String type, boolean isRoaming) {
	// Default metered apn type: [default, supl, dun, mms]
	log("isMeteredApnType, apnType = " + type + ", isRoaming = " + isRoaming);
	if (TextUtils.equals(type, PhoneConstants.APN_TYPE_DEFAULT)
		|| TextUtils.equals(type, PhoneConstants.APN_TYPE_SUPL)
		|| TextUtils.equals(type, PhoneConstants.APN_TYPE_DUN)
		|| TextUtils.equals(type, PhoneConstants.APN_TYPE_MMS)) { // 去掉MMS这一行
		return true;
	}
	return false;
}
```

```java
@Override
public boolean isMeteredApnTypeByLoad() {
	// Default return false, set to true in OP plugin if needed
	return false; //改为return true;
}
```

