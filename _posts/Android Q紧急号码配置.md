
@[toc]
##### 适用版本
Android 10(Q)及以后版本
##### 配置方法
从Android Q开始，google提供了新的紧急号码配置方法(packages/services/Telephony/ecc), 同时MTK还支持通过ecc_list.xml配置紧急号码，所以从Android Q开始可以有两种方法配置紧急号码：

**方法1：使用Google eccdata配置紧急号码(详细方法请参考packages/services/Telephony/ecc/README.md）**

>NOTE: Because we override telephony service repo, if you want to change AOSP ECC, please modify following repo: vendor/mediatek/proprietary/packages/services/Telephony
- 支持根据国家进行紧急号码配置
- 不支持根据特定运营商进行紧急号码配置
- 不支持根据地区进行进行紧急号码配置
- 不支持service category(代码里目前不会读取)
- 不支持emergency routing（配置假紧急号码）
- 不支持根据有卡、无卡配置紧急号码

**方法2：使用MTK ecc_list.xml配置紧急号码**
-  支持根据国家进行紧急号码配置
- 支持根据特定运营商进行紧急号码配置
- 支持根据地区进行进行紧急号码配置
- 支持service category(代码不支持)
- 支持emergency routing（配置假紧急号码）
- 支持根据有卡、无卡配置紧急号码

两种配置方法对比如下：
|Support status  | AOSP(eccdata) |MTK(ecc_list.xml) |
|--|--|--|
| Support customized by country |Yes  | Yes |
| Support customized by operator | No | Yes |
| Support customized by region |No  | Yes |
| Support customized by  |No  |Yes  |
| Support service category  |No  | Yes |
| Support emergency routing  |No  |Yes  |
| Support customized by with SIM/without SIM |No  |Yes  |

可以根据上面的支持程度选择合适的紧急号码配置方法。

>NOTE:Google ECC database没有经过完整的验证和测试，如果要使用必须自行验证各国紧急号码的完整和正确性。

##### 如何更新AOSP eccdata

1. 修改input/eccdata.txt

2. 更新ecc database

    1). 根目录执行source and lunch

         source build/envsetup.sh

         lunch full_xxx-eng   （xxx是project名字）

    2). cd进入到ecc的目录：

         cd vendor/mediatek/proprietary/packages/services/Telephony/ecc
    3). 执行：bash gen_eccdata.sh 
        (实测只能用bash来执行这个脚本，用sh或者直接执行脚本会有错误)
3. Make TeleService
4. Push TeleService.apk to system/priv-app/TeleService
5. Reboot device
6. run 'atest TeleServiceTests:EccDataTest#testEccDataContent'

##### 举例

###### 1. 客制化特定国家的紧急号码
方法1：修改vendor/mediatek/proprietary/packages/services/Telephony/ecc/input/eccdata.txt加入对应国家ISO的紧急号码
countries {
iso_code: "AF"
eccs {
phone_number: "119"
types: POLICE
types: FIRE
}
…
ecc_fallback: "112"
}


方法2：修改vendor/mediatek/proprietary/external/EccList/ecc_list.xml，加入对应国家MCC的紧急号码，MNC栏位必须为”FFF”或者“FF”
ex: <EccEntry Ecc="888" Category="0" Condition="1" Plmn="440 FFF"/>

###### 2. 客制化特定运营商的紧急号码
修改vendor/mediatek/proprietary/external/EccList/ecc_list.xml，加入特定运营商MCC/MNC的紧急号码，
ex: <EccEntry Ecc="888" Category="0" Condition="1" Plmn="440 01"/>

###### 3. 客制化特定大区的紧急号码
（Ex: APAC, LATAM）：把一组国家组合在一起配置减少ECC配置的数量 （Q上新增功能）

1. 定义并添加国家到区域表，多个国家MCC用‘，’分隔
```java
static Region sRegionTable[MAX_REGION_SIZE] = {
	{"APAC", "460,440,505"}, // China, Japan Australia
	{"LATAM", "724"},// Brazilian
	{"EMEA", "234"} // UK
};
```

2. 修改vendor/mediatek/proprietary/external/EccList/ecc_list.xml，加入Plmn为预定义region的紧急号码，
ex: <EccEntry Ecc="888" Category="0" Condition="1" Plmn="APAC"/>

###### 4. 客制化假紧急号码
方法2：修改vendor/mediatek/proprietary/external/EccList/ecc_list.xml，加入condition为2的紧急号码，
ex: <EccEntry Ecc="888" Category="0" Condition="2" Plmn="440 01"/>

###### 5. 客制化有卡紧急号码（无卡不是紧急号码）Q上新增功能

1. 修改vendor/mediatek/proprietary/external/EccList/ecc_list.xml，加入condition为3的紧急号码，
ex: <EccEntry Ecc="888" Category="0" Condition="3" Plmn="440 01"/>

2. <font color=red>同时需要删除AOSP eccdata里的紧急号码（如果有配置相同的紧急号码）</font>

###### 6. 客制化无卡紧急号码（有卡不是紧急号码）
1. 修改vendor/mediatek/proprietary/external/EccList/ecc_list.xml，加入condition为0的紧急号码，
ex: <EccEntry Ecc="888" Category="0" Condition="0" Plmn="440 01"/>
2. <font color=red>同时需要删除AOSP eccdata里的紧急号码（如果有配置相同的紧急号码）</font>
##### EccEntry条目解释
```java
<EccEntry Ecc="888" Category="0" Condition="0" Plmn="440 01"/>
```
1. Ecc：紧急号码
2. Category: 服务类别 (参考 3GPP TS24.008) （默认是0， 如果运营商有要求请询问运营商）
- Bit 1 (1): Police
- Bit 2 (2): Ambulance
- Bit 3 (4): Fire Brigade
- Bit 4 (8): Marine Guard
- Bit 5 (16): Mountain Rescue
- Bit 6 (32): Manually initiated eCall
- Bit 7 (64): Automatically initiated eCall
- Bit 8 (128): is spare and set to "0"

3. Condition:
- 0: 只有无卡的时候才是紧急号码，有卡的时候不是紧急号码
- 1: 无卡有卡都是真紧急号码
- 2: 有卡（没有lock）：假紧急号码，界面上显示紧急拨号，但实际以普通方式拨出
无卡或者有卡（有lock）：真紧急，用紧急拨号的方式呼出

4. Plmn: 如果该紧急号码只针对特定运营商，需要指定该紧急号码所属的PLMN，比如：
- 不指定或者为""代表该紧急号码适用于所有网络
- 特定国家所有运营商的紧急号码：460 FFF (FFF代表适用于中国大陆所有的MNC的运营商)
- 特定运营商紧急号码：460 01 (只适用于中国联通)

##### 运营商特殊配置
- 中国大陆非CDMA网络（中国移动/中国联通）要求在有卡情况下110/119/120/122必须使用普通呼叫的方式才能打到人工台，如果使用紧急呼叫的方法会打到语音台（无报警能力），而在无卡情况下需要用紧急呼叫方式打到语音台，配置如下（请务必不要修改删除）：

```xml
<EccEntry Ecc="110" Category="0" Condition="2" Plmn="460 FFF"/>
<EccEntry Ecc="119" Category="0" Condition="2" Plmn="460 FFF"/>
<EccEntry Ecc="120" Category="0" Condition="2" Plmn="460 FFF"/>
<EccEntry Ecc="122" Category="0" Condition="2" Plmn="460 FFF"/>
```
- 印度紧急号码100,101,102等需要用普通呼叫的方式才能打通，如果通过紧急呼叫的方式无法接通，另外无卡情况不支持紧急呼叫。不同运营商对紧急呼叫显示要求不同，有的要求显示紧急呼叫，有的显示普通呼叫，目前默认没有配置，如果运营商要求显示紧急呼叫，可以进行如下配置：

```xml
<EccEntry Ecc="100" Category="0" Condition="2" Plmn="404 FFF"/>
<EccEntry Ecc="101" Category="0" Condition="2" Plmn="404 FFF"/>
<EccEntry Ecc="102" Category="0" Condition="2" Plmn="404 FFF"/>
```
##### 常见问题
- Q：XML里没有配置紧急号码，为什么UI会显示紧急呼叫？
A：从Android N以后，会通过google libphonenuber判断本地紧急号码，如需修改请参考：[如何修改本地紧急号码规则（Local Emergency Number）](https://blog.csdn.net/wcsbhwy/article/details/106567190)

