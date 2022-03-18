##### [DESCRIPTION]
MTK默认的版本中，Setting里面的增强4GLTE开关（VoLTE）默认是开启的，根据运营商的定制需求，想要将其默认设置为关闭状态

##### [SOLUTION]
(1)
package com.android.providers.settings;

DatabaseHelper.java

loadSetting(stmt, Settings.Global.ENHANCED_4G_MODE_ENABLED, ImsConfig.FeatureValueConstants.ON);（两处地方），将ImsConfig.FeatureValueConstants.ON 改为 OFF

 

（2）

alps/device/mediatek/common/device.mk 文件中如下位置

ifeq ($(strip $(MTK_VOLTE_SUPPORT)), yes)

PRODUCT_PROPERTY_OVERRIDES += ro.mtk_volte_support=1

PRODUCT_PROPERTY_OVERRIDES += persist.mtk.volte.enable=1

endif

将persist.mtk.volte.enable=1

修改为 ：persist.mtk.volte.enable=0
