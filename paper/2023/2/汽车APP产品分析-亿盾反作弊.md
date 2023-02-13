# 汽车APP产品分析-亿盾反作弊
```css
目录:
一、概述
1.1、风控概述
1.2、工作流程
二、整体框架
三、初始化逻辑
四、环境检测与设备指纹
五、算法分析
六、协议还原
七、总结
```

### **一、概述**

**1.1、风控概述**

几乎所有的APP运营人员都会接触到渠道推广。这些渠道推广可能是付费渠道，可能是免费渠道，无论是哪一种渠道推广，都是需要我们付出成本的。在与渠道打交道的过程中，有时候涉及到跟渠道分成或者跟渠道合作，我们需要统计从渠道获取的用户的数量；有时候涉及到渠道付费，我们需要鉴别渠道用户的质量的好坏，控制并提高渠道的效果。

可能存在渠道都投放了，点击量也特别高，但激活量只有个位数。也有可能点击激活数量都很高，但是留存率很低。费用都花光了，但是效果没有出来。

有流量的地方就有作弊，渠道可以通过批量机器或模拟器模拟下载，以及通过人工或者技术手段修改设备信息、破解反作弊SDK方式发送虚拟信息、模拟下载激活APP等等

目前主流的反作弊方式为企业自研或使用第三方反作弊服务，APP端集成SDK，通过用户上网设备的硬件、网络、环境等设备特征信息生成设备指纹， 生成可抗黑产破解的设备唯一标识。作为纵深防御风控体系下的重要工具，可实现对终端设备上的风险环境识别、风险检测及行为风险分析。

**1.2、工作流程**

设备反作弊服务时的调用过程如图1-1所示：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/f6d5d523-009f-43b4-85a6-3a55bab0f3a0.png?raw=true)

                                图1-1  

业务客户端需要集成反作弊客户端SDK，包括安卓，iOS，H5，小程序等；通过反作弊SDK采集的设备信息可以生成设备指纹与判断设备是否安全。

### **二、整体框架**

反作弊SDK通过采集设备的硬件、网络、环境等特征信息生成设备的唯一标识，可以侦测模拟器、刷机改机、root越狱、劫持注入等风险，配合风控策略，可以对抗设备伪造改机、自动注册、渠道作弊行为。整体框架如图2-1所示：

![](https://github.com/D0n9/paper_archive/blob/main/paper/picture/2023/2/707a43e8-9cf7-467f-a1ae-c5e5f8dbd08a.jpeg?raw=true)

                                图2-1  

### **三、初始化逻辑**

**3.1、init接口**

init：初始化SDK

getToken：调用getToken接口获取token，每次调用返回不同的token值。一般由用户操作（提交等）来触发getToken，一个token只能使用一次，如果在同一个页面中，允许用户多次操作，请多次调用getToken接口以获取不同的token；提交token：将token作为请求参数提交到业务方后端。

**3.2、静态与动态**

采集设备信息分为静态与动态分别上报。

```kotlin
private a c(String arg7) {
    JSONObject v7;
String v0 = "";
int v1 = 0;
try {
    String v2 = this.a_deviceinfo(arg7);
    if(v2 != null) {
        if(v2.isEmpty()) {
            return null;
        }

        String v3 = this.d ? com.netease.mobsec.e.e.a() + "/v2/m/b" : f.u.k();
        if(v3.isEmpty()) {
            v3 = "https://ac.dun.163yun.com/v2/m/b";
        }

        JSONObject v2_1 = com.netease.mobsec.e.e.a(v3, v2, this.e);
        if(v2_1 == null) {
            v7 = com.netease.mobsec.e.e.a(v3, this.a_deviceinfo(arg7), this.e);
            if(v7 == null) {
                return new a(v1, v0);
            }
        }
        else {
            v1 = f.u.b_parsejson(v2_1);
            v0 = v2_1.optString("msg");
            if(v1 != 420) {
                return new a(v1, v0);
            }
```

```kotlin
private int a() {
    try {
        String v2 = this.b_deviceinfo("");
        if(TextUtils.isEmpty(v2)) {
            return 0;
        }

        String v3 = this.d ? com.netease.mobsec.e.e.a() + "/v2/m/d" : f.u.m();
        if(v3.isEmpty()) {
            v3 = "https://ac.dun.163yun.com/v2/m/d";
        }

        int v4 = 3000;
        JSONObject v2_1 = com.netease.mobsec.e.e.a(v3, v2, 3000);
        if(this.e != 0) {
            v4 = this.e;
        }

        if(v2_1 == null) {
            String v0_2 = this.b_deviceinfo("");
            if(v0_2 == null) {
                return 0;
            }

            JSONObject v0_3 = com.netease.mobsec.e.e.a(v3, v0_2, v4);
            if(v0_3 != null) {
                int v0_4 = f.u.b_parsejson(v0_3);
                if(v0_4 == 200) {
                    this.b();
                    return v0_4;
                }
            }
        }
        else {
            int v2_2 = f.u.b_parsejson(v2_1);
```

**3.3、注册native方法**

so层字符串xor加密

```makefile
.text:000000775DC53154 0C 7D C9 9B             UMULH           X12, X8, X9
.text:000000775DC53158 0E 01 0C CB             SUB             X14, X8, X12
.text:000000775DC5315C 8C 05 4E 8B             ADD             X12, X12, X14,LSR#1
.text:000000775DC53160 8C FD 44 D3             LSR             X12, X12, #4
.text:000000775DC53164 8C 5D 0A 9B             MADD            X12, X12, X10, X23
.text:000000775DC53168 ED 02 08 8B             ADD             X13, X23, X8
.text:000000775DC5316C 8C 01 08 8B             ADD             X12, X12, X8
.text:000000775DC53170 AD 99 40 39             LDRB            W13, [X13,#0x26]
.text:000000775DC53174 8C 45 40 39             LDRB            W12, [X12,#0x11]
.text:000000775DC53178 8C 01 0D 4A             EOR             W12, W12, W13
.text:000000775DC5317C 6C 69 28 38             STRB            W12, [X11,X8]
.text:000000775DC53180 08 05 00 91             ADD             X8, X8, #1    ; com/netease/mobsec/factory/JNIFactor
.text:000000775DC53184 1F 95 00 F1             CMP             X8, #0x25 ; '%'
 .text:000000775DC53188 61 FE FF 54             B.NE            DecString_loc_7C91120154
```

反调试检测TracerPid:

RegisterNatives 注册native方法

```makefile
.text:000000775DC551D4 68 02 40 F9             LDR             X8, [X19]
.text:000000775DC551D8 E9 07 00 32             MOV             W9, #3
.text:000000775DC551DC 23 01 15 4B             SUB             W3, W9, W21
.text:000000775DC551E0 08 5D 43 F9             LDR             X8, [X8,#0x6B8]
.text:000000775DC551E4                         ;   try {
.text:000000775DC551E4 E2 07 00 90 42 60 19 91 ADRL            X2, qword_775DD51658
.text:000000775DC551EC E0 03 13 AA             MOV             X0, X19
.text:000000775DC551F0 E1 03 14 AA             MOV             X1, X20
.text:000000775DC551F4 00 01 3F D6             BLR             X8            ; RegisterNatives
.text:000000775DC551F4                         ;   } // starts at 775DC551E4
.text:000000775DC551F4
.text:000000775DC551F8 F3 03 00 2A             MOV             W19, W0
注册方法
w238jfd9349jdj394
w230921e1b36f7799
```

### **四、环境检测与设备指纹**

**4.1、设备采集**

获取MAC

重要的设备信息通过SVC 0 方式与system_property获取:

```makefile
wifi.interface
_system_property_find
.text:0000006C146ADF74
.text:0000006C146ADF74                         system_property_find_sub_6F738E0F74
.text:0000006C146ADF74
.text:0000006C146ADF74                         s= -0x78
    .text:0000006C146ADF74                         var_68= -0x68
    .text:0000006C146ADF74                         var_58= -0x58
    .text:0000006C146ADF74                         var_48= -0x48
    .text:0000006C146ADF74                         var_38= -0x38
    .text:0000006C146ADF74                         var_28= -0x28
    .text:0000006C146ADF74                         var_20= -0x20
    .text:0000006C146ADF74                         var_18= -0x18
    .text:0000006C146ADF74                         var_10= -0x10
    .text:0000006C146ADF74                         var_s0=  0
    .text:0000006C146ADF74
.text:0000006C146ADF74                         ; __unwind { // __gxx_personality_v0
.text:0000006C146ADF74 FF 43 02 D1             SUB             SP, SP, #0x90
.text:0000006C146ADF78 F4 4F 07 A9             STP             X20, X19, [SP,#0x80+var_10]
.text:0000006C146ADF7C FD 7B 08 A9             STP             X29, X30, [SP,#0x80+var_s0]
.text:0000006C146ADF80 FD 03 02 91             ADD             X29, SP, #0x80
.text:0000006C146ADF84 54 D0 3B D5             MRS             X20, #3, c13, c0, #2
.text:0000006C146ADF88 89 16 40 F9             LDR             X9, [X20,#0x28]
.text:0000006C146ADF8C F3 03 08 AA             MOV             X19, X8
.text:0000006C146ADF90 A9 83 1E F8             STUR            X9, [X29,#var_18]
.text:0000006C146ADF94 21 00 40 F9             LDR             X1, [X1]
.text:0000006C146ADF98 29 80 5E F8             LDUR            X9, [X1,#-0x18]
.text:0000006C146ADF9C 89 02 00 B4             CBZ             X9, loc_6C146ADFEC
.text:0000006C146ADF9C
.text:0000006C146ADFA0 00 E4 00 6F             MOVI            V0.2D, #0
.text:0000006C146ADFA4 FF 63 00 B9             STR             WZR, [SP,#0x80+var_20]
.text:0000006C146ADFA8 FF 2F 00 F9             STR             XZR, [SP,#0x80+var_28]
.text:0000006C146ADFAC E0 83 84 3C             STUR            Q0, [SP,#0x80+var_38]
.text:0000006C146ADFB0 E0 83 83 3C             STUR            Q0, [SP,#0x80+var_48]
.text:0000006C146ADFB4 E0 83 82 3C             STUR            Q0, [SP,#0x80+var_58]
.text:0000006C146ADFB8 E0 83 81 3C             STUR            Q0, [SP,#0x80+var_68]
.text:0000006C146ADFBC E0 83 80 3C             STUR            Q0, [SP,#0x80+s]
.text:0000006C146ADFC0 08 00 40 F9             LDR             X8, [X0]
.text:0000006C146ADFC4 E2 23 00 91             ADD             X2, SP, #0x80+s
.text:0000006C146ADFC8 08 29 40 F9             LDR             X8, [X8,#0x50]
.text:0000006C146ADFCC 00 01 3F D6             BLR             X8            ; _system_property_find
.text:0000006C146ADFCC
.text:0000006C146ADFD0 1F 04 00 71             CMP             W0, #1
.text:0000006C146ADFD4 8B 01 00 54             B.LT            loc_6C146AE004
.text:0000006C146ADFD4
.text:0000006C146ADFD8                         ;   try {
.text:0000006C146ADFD8 E1 23 00 91             ADD             X1, SP, #0x80+s ; s
.text:0000006C146ADFDC E2 03 00 91             MOV             X2, SP
.text:0000006C146ADFE0 E0 03 13 AA             MOV             X0, X19       ; int
.text:0000006C146ADFE4 4C D5 00 94             BL              strlen_new_sub_77982AB514
.text:0000006C146ADFE4                         ;   } // starts at 6C146ADFD8
.text:0000006C146ADFE4
.text:0000006C146ADFE8 0C 00 00 14             B               loc_6C146AE018
.text:0000006C146ADFE8
.text:0000006C146ADFEC
.text:0000006C146ADFEC                         loc_6C146ADFEC
.text:0000006C146ADFEC                         ;   try {                     ; s
.text:0000006C146ADFEC 81 02 00 90 21 0C 13 91 ADRL            X1, (aBasicStringAtN+0x43)
.text:0000006C146ADFF4 E2 23 00 91             ADD             X2, SP, #0x80+s
.text:0000006C146ADFF8 E0 03 13 AA             MOV             X0, X19       ; int
.text:0000006C146ADFFC 46 D5 00 94             BL              strlen_new_sub_77982AB514
.text:0000006C146ADFFC                         ;   } // starts at 6C146ADFEC
.text:0000006C146ADFFC
.text:0000006C146AE000 06 00 00 14             B               loc_6C146AE018
.text:0000006C146AE000
.text:0000006C146AE004
.text:0000006C146AE004                         loc_6C146AE004 
.text:0000006C146AE004                         ;   try {                     ; s
.text:0000006C146AE004 61 02 00 F0 21 0C 13 91 ADRL            X1, (aBasicStringAtN+0x43)
.text:0000006C146AE00C E2 03 00 91             MOV             X2, SP
.text:0000006C146AE010 E0 03 13 AA             MOV             X0, X19       ; int
.text:0000006C146AE014 40 D5 00 94             BL              strlen_new_sub_77982AB514
.text:0000006C146AE014                         ;   } // starts at 6C146AE004
.text:0000006C146AE014
.text:0000006C146AE018
.text:0000006C146AE018                         loc_6C146AE018 
.text:0000006C146AE018 88 16 40 F9             LDR             X8, [X20,#0x28]
.text:0000006C146AE01C A9 83 5E F8             LDUR            X9, [X29,#var_18]
.text:0000006C146AE020 1F 01 09 EB             CMP             X8, X9
.text:0000006C146AE024 A1 00 00 54             B.NE            loc_6C146AE038
.text:0000006C146AE024
.text:0000006C146AE028 FD 7B 48 A9             LDP             X29, X30, [SP,#0x80+var_s0]
.text:0000006C146AE02C F4 4F 47 A9             LDP             X20, X19, [SP,#0x80+var_10]
.text:0000006C146AE030 FF 43 02 91             ADD             SP, SP, #0x90
.text:0000006C146AE034 C0 03 5F D6             RET

/proc/net/arp
/etc/wifi/wpa_supplicant.conf
.text:0000006C1462CE70                         svc_openat_sub_6F7385FE70
.text:0000006C1462CE70 08 07 80 D2             MOV             X8, #0x38 ; '8'
.text:0000006C1462CE74 01 00 00 D4             SVC             0
.text:0000006C1462CE78 1F 04 40 B1             CMN             X0, #1,LSL#12
.text:0000006C1462CE7C 00 94 80 DA             CNEG            X0, X0, HI
.text:0000006C1462CE80 88 2C 2F 54             B.HI            sub_6C1468B410
.text:0000006C1462CE80
```

其它设备信息主要通过反射获取。

```makefile
.text:0000006C1469FDA4 FF 43 02 D1             SUB             SP, SP, #0x90
.text:0000006C1469FDA8 FC 6F 03 A9             STP             X28, X27, [SP,#0xF0+var_C0]
.text:0000006C1469FDAC FA 67 04 A9             STP             X26, X25, [SP,#0xF0+var_B0]
.text:0000006C1469FDB0 F8 5F 05 A9             STP             X24, X23, [SP,#0xF0+var_A0]
.text:0000006C1469FDB4 F6 57 06 A9             STP             X22, X21, [SP,#0xF0+var_90]
.text:0000006C1469FDB8 F4 4F 07 A9             STP             X20, X19, [SP,#0xF0+var_80]
.text:0000006C1469FDBC FD 7B 08 A9             STP             X29, X30, [SP,#0xF0+var_70]
.text:0000006C1469FDC0 FD 03 02 91             ADD             X29, SP, #0x80
.text:0000006C1469FDC4 5C D0 3B D5             MRS             X28, #3, c13, c0, #2
.text:0000006C1469FDC8 F3 03 08 AA             MOV             X19, X8
.text:0000006C1469FDCC 88 17 40 F9             LDR             X8, [X28,#0x28]
.text:0000006C1469FDD0 F4 03 00 AA             MOV             X20, X0
.text:0000006C1469FDD4 E8 17 00 F9             STR             X8, [SP,#0xF0+var_C8]
.text:0000006C1469FDD8                         ;   try {
.text:0000006C1469FDD8 E1 02 00 D0 21 0C 13 91 ADRL            X1, (aBasicStringAtN+0x43) ; s
.text:0000006C1469FDE0 E2 43 00 91             ADD             X2, SP, #0xF0+var_E0
.text:0000006C1469FDE4 E0 03 13 AA             MOV             X0, X19       ; int
.text:0000006C1469FDE8 CB 0D 01 94             BL              strlen_new_sub_77982AB514
.text:0000006C1469FDE8                         ;   } // starts at 6C1469FDD8
.text:0000006C1469FDE8
.text:0000006C1469FDEC 69 23 8C D2 29 F7 B4 F2+MOV             X9, #0x9611A7B9611B
.text:0000006C1469FDEC 29 C2 D2 F2
.text:0000006C1469FDF8 5A 03 00 D0             ADRP            X26, #unk_6C14709C00@PAGE
.text:0000006C1469FDFC 8B 05 00 F0             ADRP            X11, #aAndroidPermiss_1@PAGE ; "android.permission.ACCESS_WIFI_STATE"
    .text:0000006C1469FE00 E8 03 1F AA             MOV             X8, XZR
.text:0000006C1469FE04 69 4F E3 F2             MOVK            X9, #0x1A7B,LSL#48
.text:0000006C1469FE08 EA F3 7B B2             MOV             X10, #0xFFFFFFFFFFFFFFE3
.text:0000006C1469FE0C 5A 03 30 91             ADD             X26, X26, #unk_6C14709C00@PAGEOFF
.text:0000006C1469FE10 6B E1 3F 91             ADD             X11, X11, #aAndroidPermiss_1@PAGEOFF ; "android.permission.ACCESS_WIFI_STATE"
    .text:0000006C1469FE10
.text:0000006C1469FE14
.text:0000006C1469FE14                         loc_6C1469FE14                ; CODE XREF: getSimCountryIso_sub_6F738D2910+538↓j
.text:0000006C1469FE14 0C 7D C9 9B             UMULH           X12, X8, X9
.text:0000006C1469FE18 0E 01 0C CB             SUB             X14, X8, X12
.text:0000006C1469FE1C 8C 05 4E 8B             ADD             X12, X12, X14,LSR#1
.text:0000006C1469FE20 8C FD 44 D3             LSR             X12, X12, #4
.text:0000006C1469FE24 8C 69 0A 9B             MADD            X12, X12, X10, X26
.text:0000006C1469FE28 4D 03 08 8B             ADD             X13, X26, X8
.text:0000006C1469FE2C 8C 01 08 8B             ADD             X12, X12, X8
.text:0000006C1469FE30 AD 01 4B 39             LDRB            W13, [X13,#0x2C0]
.text:0000006C1469FE34 8C 8D 4A 39             LDRB            W12, [X12,#0x2A3]
.text:0000006C1469FE38 8C 01 0D 4A             EOR             W12, W12, W13
.text:0000006C1469FE3C 6C 69 28 38             STRB            W12, [X11,X8]
.text:0000006C1469FE40 08 05 00 91             ADD             X8, X8, #1    ; android.permission.ACCESS_WIFI_STATE
.text:0000006C1469FE44 1F 95 00 F1             CMP             X8, #0x25 ; '%'
    .text:0000006C1469FE48 61 FE FF 54             B.NE            loc_6C1469FE14
.text:0000006C1469FE48
.text:0000006C1469FE4C                         ;   try {
.text:0000006C1469FE4C 81 05 00 F0 21 E0 3F 91 ADRL            X1, aAndroidPermiss_1 ; s
.text:0000006C1469FE54 E0 63 00 91             ADD             X0, SP, #0xF0+var_D8 ; int
.text:0000006C1469FE58 E2 83 00 91             ADD             X2, SP, #0xF0+var_D0
.text:0000006C1469FE5C AE 0D 01 94             BL              strlen_new_sub_77982AB514
.text:0000006C1469FE5C                         ;   } // starts at 6C1469FE4C
.text:0000006C1469FE5C
.text:0000006C1469FE60 88 02 40 F9             LDR             X8, [X20]
.text:0000006C1469FE64 08 5D 40 F9             LDR             X8, [X8,#0xB8]
.text:0000006C1469FE68                         ;   try {
.text:0000006C1469FE68 E1 63 00 91             ADD             X1, SP, #0xF0+var_D8
.text:0000006C1469FE6C E0 03 14 AA             MOV             X0, X20
.text:0000006C1469FE70 00 01 3F D6             BLR             X8            ; checkCallingOrSelfPermission判断是否有权限
.text:0000006C1469FE70                         ;   } // starts at 6C1469FE68
.text:0000006C1469FE70
.text:0000006C1469FE74 E8 0F 40 F9             LDR             X8, [SP,#0xF0+var_D8]
.text:0000006C1469FE78 7B 05 00 D0             ADRP            X27, #off_6C1474DF48@PAGE
.text:0000006C1469FE7C 7B A7 47 F9             LDR             X27, [X27,#off_6C1474DF48@PAGEOFF]
.text:0000006C1469FE80 F5 03 00 2A             MOV             W21, W0
.text:0000006C1469FE84 00 61 00 D1             SUB             X0, X8, #0x18 ; ptr
.text:0000006C1469FE88 1F 00 1B EB             CMP             X0, X27
.text:0000006C1469FE8C C1 22 00 54             B.NE            loc_6C146A02E4
.text:0000006C1469FE8C
.text:0000006C1469FE90
.text:0000006C1469FE90                         loc_6C1469FE90                ; CODE XREF: getSimCountryIso_sub_6F738D2910+A08↓j
.text:0000006C1469FE90 35 21 00 36             TBZ             W21, #0, loc_6C146A02B4
.text:0000006C1469FE90
.text:0000006C1469FE94
.text:0000006C1469FE94                         loc_6C1469FE94                ; CODE XREF: getSimCountryIso_sub_6F738D2910+A14↓j
.text:0000006C1469FE94 48 03 00 D0             ADRP            X8, #xmmword_6C147092E0@PAGE
.text:0000006C1469FE98 49 03 00 D0             ADRP            X9, #qword_6C147096D8@PAGE
.text:0000006C1469FE9C 00 B9 C0 3D             LDR             Q0, [X8,#xmmword_6C147092E0@PAGEOFF]
.text:0000006C1469FEA0 21 6D 43 FD             LDR             D1, [X9,#qword_6C147096D8@PAGEOFF]
.text:0000006C1469FEA4 80 0A 40 F9             LDR             X0, [X20,#0x10]
.text:0000006C1469FEA8 A1 05 00 90 21 80 00 91 ADRL            X1, aAndroidContent_0 ; "android/content/Context"
    .text:0000006C1469FEB0 20 00 80 3D             STR             Q0, [X1]      ; "android/content/Context"
    .text:0000006C1469FEB4 21 08 00 FD             STR             D1, [X1,#(aAndroidContent_0+0x10 - 0x6C14753020)] ; "Context"
    .text:0000006C1469FEB8 08 00 40 F9             LDR             X8, [X0]
.text:0000006C1469FEBC 08 19 40 F9             LDR             X8, [X8,#0x30]
.text:0000006C1469FEC0                         ;   try {
.text:0000006C1469FEC0 00 01 3F D6             BLR             X8            ; FindClass
.text:0000006C1469FEC0                         ;   } // starts at 6C1469FEC0
.text:0000006C1469FEC0
.text:0000006C1469FEC4 F5 03 00 AA             MOV             X21, X0
.text:0000006C1469FEC8 60 1F 00 B4             CBZ             X0, loc_6C146A02B4
.text:0000006C1469FEC8
.text:0000006C1469FECC 49 03 00 D0             ADRP            X9, #xmmword_6C147092F0@PAGE
.text:0000006C1469FED0 20 BD C0 3D             LDR             Q0, [X9,#xmmword_6C147092F0@PAGEOFF]
.text:0000006C1469FED4 80 0A 40 F9             LDR             X0, [X20,#0x10]
.text:0000006C1469FED8 AB 05 00 90             ADRP            X11, #aGetsystemservi@PAGE ; "getSystemService"
    .text:0000006C1469FEDC A9 D8 89 D2             MOV             X9, #0x4EC5
.text:0000006C1469FEE0 6B 01 01 91             ADD             X11, X11, #aGetsystemservi@PAGEOFF ; "getSystemService"
    .text:0000006C1469FEE4 89 9D B8 F2             MOVK            X9, #0xC4EC,LSL#16
.text:0000006C1469FEE8 C9 89 DD F2             MOVK            X9, #0xEC4E,LSL#32
.text:0000006C1469FEEC 7F 41 00 39             STRB            WZR, [X11,#(aGetsystemservi+0x10 - 0x6C14753040)] ; ""
    .text:0000006C1469FEF0 60 01 80 3D             STR             Q0, [X11]     ; "getSystemService"
    .text:0000006C1469FEF4 AB 05 00 90             ADRP            X11, #aLjavaLangStrin_1@PAGE ; "(Ljava/lang/String;)Ljava/lang/Object;"
    .text:0000006C1469FEF8 E8 03 1F AA             MOV             X8, XZR
.text:0000006C1469FEFC 89 D8 E9 F2             MOVK            X9, #0x4EC4,LSL#48
.text:0000006C1469FF00 2A 03 80 92             MOV             X10, #0xFFFFFFFFFFFFFFE6
.text:0000006C1469FF04 6B 45 01 91             ADD             X11, X11, #aLjavaLangStrin_1@PAGEOFF ; "(Ljava/lang/String;)Ljava/lang/Object;"
    .text:0000006C1469FF04
.text:0000006C1469FF08
.text:0000006C1469FF08                         loc_6C1469FF08                ; CODE XREF: getSimCountryIso_sub_6F738D2910+624↓j
.text:0000006C1469FF08 0C 7D C9 9B             UMULH           X12, X8, X9
.text:0000006C1469FF0C 8C FD 43 D3             LSR             X12, X12, #3
.text:0000006C1469FF10 8C 69 0A 9B             MADD            X12, X12, X10, X26
.text:0000006C1469FF14 4D 03 08 8B             ADD             X13, X26, X8
.text:0000006C1469FF18 8C 01 08 8B             ADD             X12, X12, X8
.text:0000006C1469FF1C AD 3D 4E 39             LDRB            W13, [X13,#0x38F]
.text:0000006C1469FF20 8C D5 4D 39             LDRB            W12, [X12,#0x375]
.text:0000006C1469FF24 8C 01 0D 4A             EOR             W12, W12, W13
.text:0000006C1469FF28 6C 69 28 38             STRB            W12, [X11,X8] ; (Ljava/lang/String;)Ljava/lang/Object;
  .text:0000006C1469FF2C 08 05 00 91             ADD             X8, X8, #1
.text:0000006C1469FF30 1F 9D 00 F1             CMP             X8, #0x27 ; '''
    .text:0000006C1469FF34 A1 FE FF 54             B.NE            loc_6C1469FF08
.text:0000006C1469FF34
.text:0000006C1469FF38 08 00 40 F9             LDR             X8, [X0]
.text:0000006C1469FF3C 08 85 40 F9             LDR             X8, [X8,#0x108]
.text:0000006C1469FF40                         ;   try {
.text:0000006C1469FF40 A2 05 00 90             ADRP            X2, #aGetsystemservi@PAGE ; "getSystemService"
.text:0000006C1469FF44 A3 05 00 90             ADRP            X3, #aLjavaLangStrin_1@PAGE ; "(Ljava/lang/String;)Ljava/lang/Object;"
.text:0000006C1469FF48 42 00 01 91             ADD             X2, X2, #aGetsystemservi@PAGEOFF ; "getSystemService"
.text:0000006C1469FF4C 63 44 01 91             ADD             X3, X3, #aLjavaLangStrin_1@PAGEOFF ; "(Ljava/lang/String;)Ljava/lang/Object;"
.text:0000006C1469FF50 E1 03 15 AA             MOV             X1, X21
.text:0000006C1469FF54 00 01 3F D6             BLR             X8            ; GetMethodID
.text:0000006C1469FF54                         ;   } // starts at 6C1469FF40
.text:0000006C1469FF54
.text:0000006C1469FF58 F7 03 00 AA             MOV             X23, X0
.text:0000006C1469FF5C 20 1A 00 B4             CBZ             X0, loc_6C146A02A0
.text:0000006C1469FF5C
.text:0000006C1469FF60 48 03 00 D0             ADRP            X8, #qword_6C147096E0@PAGE
.text:0000006C1469FF64 00 71 43 FD             LDR             D0, [X8,#qword_6C147096E0@PAGEOFF]
.text:0000006C1469FF68 A2 05 00 90 42 E0 01 91 ADRL            X2, aWifiService ; "WIFI_SERVICE"
    .text:0000006C1469FF70 48 03 00 D0             ADRP            X8, #xmmword_6C14709300@PAGE
.text:0000006C1469FF74 80 0A 40 F9             LDR             X0, [X20,#0x10]
.text:0000006C1469FF78 40 00 00 FD             STR             D0, [X2]      ; "WIFI_SERVICE"
    .text:0000006C1469FF7C 00 C1 C0 3D             LDR             Q0, [X8,#xmmword_6C14709300@PAGEOFF]
.text:0000006C1469FF80 C9 2A 89 52             MOV             W9, #0x4956
.text:0000006C1469FF84 A3 05 00 90             ADRP            X3, #xmmword_6C14753090@PAGE
.text:0000006C1469FF88 69 A8 A8 72             MOVK            W9, #0x4543,LSL#16
.text:0000006C1469FF8C 63 40 02 91             ADD             X3, X3, #xmmword_6C14753090@PAGEOFF
.text:0000006C1469FF90 E8 6C 87 52             MOV             W8, #0x3B67
.text:0000006C1469FF94 49 08 00 B9             STR             W9, [X2,#(aWifiService+8 - 0x6C14753078)] ; "VICE"
    .text:0000006C1469FF98 5F 30 00 39             STRB            WZR, [X2,#(aWifiService+0xC - 0x6C14753078)] ; ""
    .text:0000006C1469FF9C 68 20 00 79             STRH            W8, [X3,#(word_6C147530A0 - 0x6C14753090)]
.text:0000006C1469FFA0 60 00 80 3D             STR             Q0, [X3]
.text:0000006C1469FFA4 7F 48 00 39             STRB            WZR, [X3,#(byte_6C147530A2 - 0x6C14753090)]
.text:0000006C1469FFA8 08 00 40 F9             LDR             X8, [X0]
.text:0000006C1469FFAC 08 41 42 F9             LDR             X8, [X8,#0x480]
.text:0000006C1469FFB0                         ;   try {
.text:0000006C1469FFB0 E1 03 15 AA             MOV             X1, X21       ; WIFI_SERVICE
.text:0000006C1469FFB4 00 01 3F D6             BLR             X8            ; GetStaticFieldID
.text:0000006C1469FFB4                         ;   } // starts at 6C1469FFB0
.text:0000006C1469FFB4
.text:0000006C1469FFB8 E2 03 00 AA             MOV             X2, X0
.text:0000006C1469FFBC 20 17 00 B4             CBZ             X0, loc_6C146A02A0
.text:0000006C1469FFBC
.text:0000006C1469FFC0 80 0A 40 F9             LDR             X0, [X20,#0x10]
.text:0000006C1469FFC4 08 00 40 F9             LDR             X8, [X0]
.text:0000006C1469FFC8 08 45 42 F9             LDR             X8, [X8,#0x488]
.text:0000006C1469FFCC                         ;   try {
.text:0000006C1469FFCC E1 03 15 AA             MOV             X1, X21
.text:0000006C1469FFD0 00 01 3F D6             BLR             X8            ; GetStaticObjectField
.text:0000006C1469FFD0                         ;   } // starts at 6C1469FFCC
.text:0000006C1469FFD0
.text:0000006C1469FFD4 F6 03 00 AA             MOV             X22, X0
.text:0000006C1469FFD8 81 82 40 A9             LDP             X1, X0, [X20,#8]
.text:0000006C1469FFDC                         ;   try {
.text:0000006C1469FFDC E2 03 17 AA             MOV             X2, X23
.text:0000006C1469FFE0 E3 03 16 AA             MOV             X3, X22
.text:0000006C1469FFE4 D8 B7 FF 97             BL              CallObjectMethodV_sub_6F6549FF44
.text:0000006C1469FFE4                         ;   } // starts at 6C1469FFDC
.text:0000006C1469FFE4
.text:0000006C1469FFE8 F7 03 00 AA             MOV             X23, X0
.text:0000006C1469FFEC 00 15 00 B4             CBZ             X0, loc_6C146A028C
.text:0000006C1469FFEC
.text:0000006C1469FFF0 80 0A 40 F9             LDR             X0, [X20,#0x10]
.text:0000006C1469FFF4 E9 71 9C D2 09 C7 B1 F2+MOV             X9, #0x38E38E38E38F
.text:0000006C1469FFF4 69 1C C7 F2
.text:0000006C146A0000 8B 05 00 F0             ADRP            X11, #aAndroidNetWifi@PAGE ; ""
.text:0000006C146A0004 E8 03 1F AA             MOV             X8, XZR
.text:0000006C146A0008 C9 71 FC F2             MOVK            X9, #0xE38E,LSL#48
.text:0000006C146A000C 2A 02 80 92             MOV             X10, #0xFFFFFFFFFFFFFFEE
.text:0000006C146A0010 6B 8D 02 91             ADD             X11, X11, #aAndroidNetWifi@PAGEOFF ; ""
```

**4.2、环境风险检测**

检测xposed，查找进程模块。

```makefile
.text:0000006C134C64A8 E8 1F 40 F9             LDR             X8, [SP,#0x2D0+var_298]
.text:0000006C134C64AC C9 05 00 B0             ADRP            X9, #xmmword_6C1357F370@PAGE
.text:0000006C134C64B0 20 DD C0 3D             LDR             Q0, [X9,#xmmword_6C1357F370@PAGEOFF]
.text:0000006C134C64B4 40 08 00 90             ADRP            X0, #aProcSelfMaps@PAGE ; "/proc/self/maps"
.text:0000006C134C64B8 08 0D 40 F9             LDR             X8, [X8,#0x18]
.text:0000006C134C64BC 41 08 00 90             ADRP            X1, #word_6C135CE400@PAGE
.text:0000006C134C64C0 00 C0 0F 91             ADD             X0, X0, #aProcSelfMaps@PAGEOFF ; "/proc/self/maps"
.text:0000006C134C64C4 21 00 10 91             ADD             X1, X1, #word_6C135CE400@PAGEOFF
.text:0000006C134C64C8 08 1D 40 F9             LDR             X8, [X8,#0x38]
.text:0000006C134C64CC 49 0E 80 52             MOV             W9, #0x72 ; 'r'
.text:0000006C134C64D0 00 00 80 3D             STR             Q0, [X0]      ; "/proc/self/maps"
.text:0000006C134C64D4 29 00 00 79             STRH            W9, [X1]      ; /proc/self/maps
.text:0000006C134C64D8 00 01 3F D6             BLR             X8            ; fopen
.text:0000006C134C64D8
.text:0000006C134C64DC E0 20 00 B4             CBZ             X0, loc_6C134C68F8
.text:0000006C134C6594 E8 1F 40 F9             LDR             X8, [SP,#0x2D0+var_298]
.text:0000006C134C6598 08 0D 40 F9             LDR             X8, [X8,#0x18]
.text:0000006C134C659C 08 21 40 F9             LDR             X8, [X8,#0x40]
.text:0000006C134C65A0                         ;   try {
.text:0000006C134C65A0 E0 83 01 91             ADD             X0, SP, #0x2D0+s
.text:0000006C134C65A4 E1 03 17 32             MOV             W1, #0x200
.text:0000006C134C65A8 E2 03 16 AA             MOV             X2, X22
.text:0000006C134C65AC 00 01 3F D6             BLR             X8            ; fgets
.text:0000006C134C65AC
.text:0000006C134C65B0 80 1A 00 B4             CBZ             X0, loc_6C134C6900
.text:0000006C134C65B0
.text:0000006C134C65B4 E0 83 01 91             ADD             X0, SP, #0x2D0+s ; s
.text:0000006C134C65B8 36 43 FF 97             BL              .strlen
.text:0000006C134C65D0                         DecString_loc_6C134C65D0      ; CODE XREF: check_xposed_sub_6F7387B450+1AC↓j
.text:0000006C134C65D0 09 7D D4 9B             UMULH           X9, X8, X20
.text:0000006C134C65D4 29 FD 44 D3             LSR             X9, X9, #4
.text:0000006C134C65D8 29 71 13 9B             MADD            X9, X9, X19, X28
.text:0000006C134C65DC 8A 03 08 8B             ADD             X10, X28, X8
.text:0000006C134C65E0 29 01 08 8B             ADD             X9, X9, X8
.text:0000006C134C65E4 4A 81 42 39             LDRB            W10, [X10,#0xA0]
.text:0000006C134C65E8 29 31 42 39             LDRB            W9, [X9,#0x8C]
.text:0000006C134C65EC 29 01 0A 4A             EOR             W9, W9, W10
.text:0000006C134C65F0 E9 6A 28 38             STRB            W9, [X23,X8]
.text:0000006C134C65F4 08 05 00 91             ADD             X8, X8, #1
.text:0000006C134C65F8 1F 5D 00 F1             CMP             X8, #0x17     ; de.robv.android.xposed
.text:0000006C134C65FC A1 FE FF 54             B.NE            DecString_loc_6C134C65D0
.text:0000006C134C65FC
.text:0000006C134C6600 E0 03 17 AA             MOV             X0, X23       ; s
.text:0000006C134C6604 23 43 FF 97             BL              .strlen
.text:0000006C134C6604
.text:0000006C134C6608 E3 03 00 AA             MOV             X3, X0
.text:0000006C134C660C E0 43 01 91             ADD             X0, SP, #0x2D0+var_280 ; int
.text:0000006C134C6610 E1 03 17 AA             MOV             X1, X23       ; void *
.text:0000006C134C6614 E2 03 1F AA             MOV             X2, XZR
.text:0000006C134C6618 F6 6D 02 94             BL              memcmp_sub_779C063DF0 ; de.robv.android.xposed
.text:0000006C134C6618
.text:0000006C134C661C 1F 04 00 B1             CMN             X0, #1
.text:0000006C134C6620 01 12 00 54             B.NE            loc_6C134C6860
.text:0000006C134C6620
.text:0000006C134C6624 E0 0B C0 3D             LDR             Q0, [SP,#0x2D0+var_2B0]
.text:0000006C134C6628 E8 0D 80 52             MOV             W8, #0x6F ; 'o'
.text:0000006C134C662C E0 03 18 AA             MOV             X0, X24       ; s
.text:0000006C134C6630 08 23 00 79             STRH            W8, [X24,#(aLibxposedArtSo+0x10 - 0x6C135CE420)] ; "o"
.text:0000006C134C6634 00 03 80 3D             STR             Q0, [X24]     ; "/libxposed_art.so"
.text:0000006C134C6638 16 43 FF 97             BL              .strlen
.text:0000006C134C6638
.text:0000006C134C663C E3 03 00 AA             MOV             X3, X0
.text:0000006C134C6640 E0 43 01 91             ADD             X0, SP, #0x2D0+var_280 ; int
.text:0000006C134C6644 E1 03 18 AA             MOV             X1, X24       ; void *
.text:0000006C134C6648 E2 03 1F AA             MOV             X2, XZR
.text:0000006C134C664C E9 6D 02 94             BL              memcmp_sub_779C063DF0 ; libxposed_art.so
.text:0000006C134C664C
.text:0000006C134C6650 1F 04 00 B1             CMN             X0, #1
.text:0000006C134C6654 61 10 00 54             B.NE            loc_6C134C6860
.text:0000006C134C6654
.text:0000006C134C6658 E0 03 19 AA             MOV             X0, X25       ; s
.text:0000006C134C665C 28 03 00 FD             STR             D8, [X25]     ; "edxp.so"
.text:0000006C134C6660 0C 43 FF 97             BL              .strlen
.text:0000006C134C6660
.text:0000006C134C6664 E3 03 00 AA             MOV             X3, X0
.text:0000006C134C6668 E0 43 01 91             ADD             X0, SP, #0x2D0+var_280 ; int
.text:0000006C134C666C E1 03 19 AA             MOV             X1, X25       ; void *
.text:0000006C134C6670 E2 03 1F AA             MOV             X2, XZR
.text:0000006C134C6674 DF 6D 02 94             BL              memcmp_sub_779C063DF0 ; edxp.so
.text:0000006C134C6674
.text:0000006C134C6678 1F 04 00 B1             CMN             X0, #1
.text:0000006C134C667C 21 0F 00 54             B.NE            loc_6C134C6860
.text:0000006C134C667C
.text:0000006C134C6680 48 2E 8C 52 88 AE AC 72 MOV             W8, #0x65746172
.text:0000006C134C6688 E0 03 1A AA             MOV             X0, X26       ; s
.text:0000006C134C668C 49 03 00 FD             STR             D9, [X26]     ; "libsubstrate"
.text:0000006C134C6690 48 0B 00 B9             STR             W8, [X26,#(aLibsubstrate+8 - 0x6C135CE440)] ; "rate"
.text:0000006C134C6694 5F 33 00 39             STRB            WZR, [X26,#(aLibsubstrate+0xC - 0x6C135CE440)] ; ""
.text:0000006C134C6698 FE 42 FF 97             BL              .strlen
.text:0000006C134C6698
.text:0000006C134C669C E3 03 00 AA             MOV             X3, X0
.text:0000006C134C66A0 E0 43 01 91             ADD             X0, SP, #0x2D0+var_280 ; int
.text:0000006C134C66A4 E1 03 1A AA             MOV             X1, X26       ; void *
.text:0000006C134C66A8 E2 03 1F AA             MOV             X2, XZR
.text:0000006C134C66AC D1 6D 02 94             BL              memcmp_sub_779C063DF0 ; libsubstrate
.text:0000006C134C66AC                         ;   } // starts at 6C134C65A0
.text:0000006C134C66AC
.text:0000006C134C66B0 1F 04 00 B1             CMN             X0, #1
.text:0000006C134C66B4 A1 0D 00 54             B.NE            loc_6C134C6868
.text:0000006C134C66B4
.text:0000006C134C66B8 48 08 00 90             ADRP            X8, #qword_6C135CE3C8@PAGE
.text:0000006C134C66BC 08 E5 41 F9             LDR             X8, [X8,#qword_6C135CE3C8@PAGEOFF]
.text:0000006C134C66C0 08 81 5E F8             LDUR            X8, [X8,#-0x18]
.text:0000006C134C66C4 1F 09 00 F1             CMP             X8, #2
.text:0000006C134C66C8 68 F6 FF 54             B.HI            loc_6C134C6594
.text:0000006C134C66C8
.text:0000006C134C66CC C8 25 8C 52 08 6E AD 72 MOV             W8, #0x6B70612E
.text:0000006C134C66D4 E0 03 1B AA             MOV             X0, X27       ; s
.text:0000006C134C66D8 68 03 00 B9             STR             W8, [X27]     ; ".apk"
.text:0000006C134C66DC 7F 13 00 39             STRB            WZR, [X27,#(aApk+4 - 0x6C135CE450)] ; ""
.text:0000006C134C66E0 EC 42 FF 97             BL              .strlen
.text:0000006C134C66E0
.text:0000006C134C66E4 E3 03 00 AA             MOV             X3, X0
.text:0000006C134C66E8                         ;   try {
.text:0000006C134C66E8 E0 43 01 91             ADD             X0, SP, #0x2D0+var_280 ; int
.text:0000006C134C66EC E1 03 1B AA             MOV             X1, X27       ; void *
.text:0000006C134C66F0 E2 03 1F AA             MOV             X2, XZR
.text:0000006C134C66F4 BF 6D 02 94             BL              memcmp_sub_779C063DF0
.text:0000006C134C66F4
.text:0000006C134C66F8 1F 04 00 31             CMN             W0, #1
.text:0000006C134C66FC 81 02 00 54             B.NE            loc_6C134C674C
```

检测magisk，读取/proc/mounts判断是否有特征

```makefile
.text:0000006C134FCDE4 88 06 40 F9             LDR             X8, [X20,#8]
.text:0000006C134FCDE8 49 04 00 B0             ADRP            X9, #qword_6C13585070@PAGE
.text:0000006C134FCDEC 20 39 40 FD             LDR             D0, [X9,#qword_6C13585070@PAGEOFF]
.text:0000006C134FCDF0 A0 06 00 90             ADRP            X0, #aProcMounts@PAGE ; ""
.text:0000006C134FCDF4 08 69 40 F9             LDR             X8, [X8,#0xD0]
.text:0000006C134FCDF8 A9 CE 8D 52             MOV             W9, #0x6E75
.text:0000006C134FCDFC 00 A0 27 91             ADD             X0, X0, #aProcMounts@PAGEOFF ; ""
.text:0000006C134FCE00 89 6E AE 72             MOVK            W9, #0x7374,LSL#16
.text:0000006C134FCE04 C2 36 80 52             MOV             W2, #0x1B6
.text:0000006C134FCE08 E1 03 1F 2A             MOV             W1, WZR
.text:0000006C134FCE0C 00 00 00 FD             STR             D0, [X0]      ; ""
.text:0000006C134FCE10 09 08 00 B9             STR             W9, [X0,#(aProcMounts+8 - 0x6C135D09E8)] ; ""
.text:0000006C134FCE14 1F 30 00 39             STRB            WZR, [X0,#(aProcMounts+0xC - 0x6C135D09E8)] ; /proc/mounts
.text:0000006C134FCE18 00 01 3F D6             BLR             X8            ; openat /proc/mounts
.text:0000006C134FCEE0 88 06 40 F9             LDR             X8, [X20,#8]
.text:0000006C134FCEE4 08 31 40 F9             LDR             X8, [X8,#0x60]
.text:0000006C134FCEE8                         ;   try {
.text:0000006C134FCEE8 E0 A3 00 91             ADD             X0, SP, #0x270+s
.text:0000006C134FCEEC E1 03 15 2A             MOV             W1, W21
.text:0000006C134FCEF0 00 01 3F D6             BLR             X8            ; read
.text:0000006C134FCEF0
.text:0000006C134FCEF4 60 02 00 B4             CBZ             X0, loc_6C134FCF40
.text:0000006C134FCEF4
.text:0000006C134FCEF8 E0 A3 00 91             ADD             X0, SP, #0x270+s ; s
.text:0000006C134FCEFC E5 68 FE 97             BL              .strlen
.text:0000006C134FCEFC
.text:0000006C134FCF00 E2 03 00 AA             MOV             X2, X0
.text:0000006C134FCF04 E0 63 00 91             ADD             X0, SP, #0x270+var_258
.text:0000006C134FCF08 E1 A3 00 91             ADD             X1, SP, #0x270+s
.text:0000006C134FCF0C 90 95 01 94             BL              memmove_sub_779C06454C
.text:0000006C134FCF0C                         ;   } // starts at 6C134FCEE8
.text:0000006C134FCF0C
.text:0000006C134FCF10 E1 0B 40 F9             LDR             X1, [SP,#0x270+var_260] ; void *
.text:0000006C134FCF14 23 80 5E F8             LDUR            X3, [X1,#-0x18]
.text:0000006C134FCF14
.text:0000006C134FCF18                         ;   try {
.text:0000006C134FCF18 E0 63 00 91             ADD             X0, SP, #0x270+var_258 ; int
.text:0000006C134FCF1C E2 03 1F AA             MOV             X2, XZR
.text:0000006C134FCF20 B4 93 01 94             BL              memcmp_sub_779C063DF0
.text:0000006C134FCF20                         ;   } // starts at 6C134FCF18
.text:0000006C134FCF20
.text:0000006C134FCF24 E1 07 40 F9             LDR             X1, [SP,#0x270+var_268] ; void *
.text:0000006C134FCF28 F6 03 00 AA             MOV             X22, X0
.text:0000006C134FCF2C 23 80 5E F8             LDUR            X3, [X1,#-0x18]
.text:0000006C134FCF2C
.text:0000006C134FCF30                         ;   try {
.text:0000006C134FCF30 E0 63 00 91             ADD             X0, SP, #0x270+var_258 ; int
.text:0000006C134FCF34 E2 03 1F AA             MOV             X2, XZR
.text:0000006C134FCF38 AE 93 01 94             BL              memcmp_sub_779C063DF0
.text:0000006C134FCF38                         ;   } // starts at 6C134FCF30
.text:0000006C134FCF38
.text:0000006C134FCF3C E5 FF FF 17             B               loc_6C134FCED0
```

检测模拟器特征

```kotlin
name=51-3||1@/./init.android_x86.rc||1@/system/./lib/libhoudini.so||1@/data/./data/com.anddoes.launcher||1@/data/./data/com.bluestacks.settings name=Ludashi||1@/init.ludashi.rc||1@/system/bin/ludashi-prop||1@/system/lib/arm/libhoudini.so name=TencentAo1||1@/init.vbox86.rc||1@/dev/socket/genyd||2@ro.product.brand: tencent||1@/system/lib/libhoudini_408p.so name=lgshouyou||1@/system/lib/libc_malloc_debug_qemu.so||1@/system/lib/libldutils.so||1@/init.android_x86.rc||2@init.svc.ldinit:  name=TencentAo2||1@/init.vbox.rc||1@/dev/socket/genyd||2@ro.product.brand: tencent||1@/system/lib/libhoudini_408p.so name=Netease1||1@/init.x86.rc||2@nemud.vt.status: ||2@nemud.device.id: ||1@/sys/kernel/debug/x86||1@/sys/module/nemusf name=LeiDian4||2@init.svc.ldinit: ||1@/system/lib/libldutils.so||1@/system/lib/arm/libhoudini.so name=Leyou||1@/init.vbox2345_x86.rc||1@/system/bin/lybox_sf||1@/system/lib/liblybox_prop.so||1@/data/data/com.emulator.app.market name=TianTian2||1@/fstab.ttVM_x86||1@/system/bin/ttVM-vbox-sf||3@com.tiantian.ime name=Netease2||1@/init.cancro.rc||2@nemud.vt.status: ||2@nemud.device.id: ||1@/sys/kernel/debug/x86||1@/sys/module/nemusf name=LeiDian5||1@/init.android_x86.rc||2@init.svc.ldinit: ||2@ro.hardware.gps: ld||5@/system/lib/arm/libhoudini.so name=TencentAo3||5@libhoudini_415c.so name=LeiDian6||2@init.svc.ldinit: ||2@ro.hardware.gps: ld||5@/system/lib/arm/libhoudini.so name=Peak||1@/init.androidVM_x86.rc||2@init.svc.pkVM_x86-setup: ||1@/data/data/com.pk.peak.launcher3 name=NOX6079||1@/system/./bin/nox||1@/system/./lib/libnoxd.so||1@/system/./lib/libhoudini.so name=xDroid||2@init.svc.xdroidd: ||2@ro.product.name: xdroid||2@ro.xdroid: ||5@/system/lib/libhoudini.so name=Sina||1@/system/lib/libc_malloc_debug_qemu.so||1@/init.x86.rc||1@/x86.prop||1@/data/data/org.adwfreak.launcher name=BlueStacks255||1@/./init.x86.rc||1@/data/./data/com.bluestacks.appmart||1@/data/./data/com.bluestacks.settings name=XiaoYao5119||1@/init.intel.rc||1@/data/data/com.microvirt.launcher||1@/data/data/com.microvirt.market name=TianTian1||1@/system/bin/ttVM-prop||1@/init.ttVM_x86.rc||2@init.svc.ttVM_x86-setup: ||1@/data/data/com.tiantian.ime name=LeiDian1||1@/system/lib/libc_malloc_debug_qemu.so||1@/system/lib/libldutils.so||1@/init.android_x86.rc||2@init.svc.ldinit: ||3@com.android.flysilkworm name=ChangWan||1@/system/bin/androVM-prop||1@/dev/socket/genyd||1@/init.vbox86p.rc||2@init.svc.vbox86-setup: ||1@/data/data/cn.itools.vm.launcher name=YZZ||1@/init.x86.rc||1@/x86.prop||1@/data/data/cn.yzz.app.launcher name=QingTing||1@/init.x86.rc||1@/x86.prop||1@/data/data/com.pop.store name=DuoWan||1@/init.x86.rc||1@/x86.prop||1@/data/data/com.duowan.coreserver name=9981||1@/system/lib/libdroidbox-ril.so||1@/system/bin/droidbox-prop||1@/init.droidbox.rc||2@ro.kernel.droidbox: ||1@/data/data/com.droidbox.market name=StartLight||1@/system/bin/droid4x-prop||1@/system/lib/libdroid4x.so||1@/init.vbox86.rc||2@init.svc.droid4x: ||1@/data/data/com.starlight.helper.task name=KaopuBlueStacks||1@/system/lib/libKaopuSdk.so||1@/init.x86.rc||1@/x86.prop||1@/data/data/com.kapou.launcher name=KaopuTianTian||1@/system/bin/ttVM-prop||1@/init.ttVM_x86.rc||1@/system/lib/libKaopuSdk.so||2@init.svc.ttVM_x86-setup: ||1@/data/data/com.kapou.launcher name=51||1@/x86.prop||1@/init.x86.rc||2@ro.product.oem: 51mnq||1@/data/data/com.anddoes.launcher name=Nox||2@init.svc.noxd:  name=XiaoYao||1@/system/bin/microvirt-prop||1@/system/lib/libmicrovirt.so||2@init.svc.vbox86-setup: ||2@init.svc.microvirtd: ||1@/data/data/com.microvirt.launcher name=Netease3||1@/system/bin/nemuVM-prop||1@/x86.prop||2@init.svc.nemu-service: ||1@/data/data/com.netease.nemu_android_launcher.nemu name=TencentAo4||1@/init.vbox86.rc||1@/dev/socket/genyd||3@com.tencent.tinput name=DDZS||1@/system/lib/libc_malloc_debug_qemu.so-arm||1@/init.x86.rc||1@/x86.prop||1@/data/data/com.kapou.launcher||1@/data/data/com.ddzs.mkt name=Droid4X||1@/system/bin/droid4x-prop||1@/system/lib/libdroid4x.so||2@init.svc.droid4x: ||1@/data/data/me.haima.androidassist name=Phoenix9||1@/init.android_x86_64.rc||1@/system/lib/libhoudini.so||1@/data/data/com.chaozhuo.filemanager.phoenixos 
name=LeiDian3||1@/system/lib/libl
```

**4.3、采集后设备信息**

```json
{
  "0": "200",
  "1": "A4465",
  "10": "",
  "2": "",
  "3": "",
  "4": "74601ae234884d9b8b5fb16501e82f18",
  "5": "1673253393",
  "6": "0",
  "7": "",
  "8": "",
  "9": "YD00458597415740",
  "A": 1673247344,
  "ABS": "{\n\t\"i\":null,\n\t\"s\":0\n}\n",
  "B": 33,
  "BD": "redfin-user13TQ1A2301050019292298release-keys",
  "BF": "google/redfin/redfin:13/TQ1A230105001/9292298:user/release-keys",
  "BP": "redfin",
  "C": "",
  "D": "r3-05-9150479",
  "F": 8,
  "GSS": "ABSENT",
  "GVB": "g7250-00220-221017-B-9183951",
  "HW": "redfin",
  "I": "4742102e42cff5c8",
  "M": 20,
  "P": "",
  "PB": "redfin",
  "PD": "redfin",
  "PN": "redfin",
  "VD": "902addd5a735a7a7395bac50911e135aac3aaad3b22c04b75272460d77073fee",
  "a": "Pixel5",
  "b": "arm64-v8a",
  "bl": 100,
  "bn": "27",
  "bs": "02:00:00:00:00:00",
  "c": "1080*2340",
  "ch": "",
  "ci": "",
  "cp": "7(QualcommTechnologies,IncLITO)",
  "cs": 1,
  "d": "c635270fada4c350dba395a6b15fec67",
  "e": "",
  "f": 19573,
  "fd": "103",
  "fm": "312688",
  "g": "4190",
  "h": 2213303839,
  "h2": 2050751049,
  "hf": "androidosBinderProxy",
  "i": "",
  "icc": "",
  "inf": 0,
  "k": "unknown",
  "l": "",
  "ll": "zh",
  "m": "02:00:00:00:00:00",
  "mf": "Google",
  "mfk": "null\n",
  "mo": 0,
  "mr": 10242,
  "n": -1,
  "nt": 1,
  "p": "",
  "pn": "comxiaopengmycarinfo",
  "q": 1673247344,
  "r": 1,
  "rm": 0,
  "s": "",
  "si": [{
    "n": "LSM6DSRAccelerometer",
    "t": 1,
    "v": "STMicro"
  }],
  "sin": "None",
  "ss": "<unknownssid>",
  "t": 5,
  "ta": "release-keys",
  "td": "109",
  "tm": "7640596",
  "u": 0,
  "v": "13",
  "vp": "",
  "wp": "",
  "y": 1673250180,
  "z": ""
}
```

### **五、算法分析**

**5.1、生成随机AES KEY**

```makefile
.text:0000006C14667914 FF 43 02 D1             SUB             SP, SP, #0x90
.text:0000006C14667918 FA 67 04 A9             STP             X26, X25, [SP,#0x80+var_40]
.text:0000006C1466791C F8 5F 05 A9             STP             X24, X23, [SP,#0x80+var_30]
.text:0000006C14667920 F6 57 06 A9             STP             X22, X21, [SP,#0x80+var_20]
.text:0000006C14667924 F4 4F 07 A9             STP             X20, X19, [SP,#0x80+var_10]
.text:0000006C14667928 FD 7B 08 A9             STP             X29, X30, [SP,#0x80+var_s0]
.text:0000006C1466792C FD 03 02 91             ADD             X29, SP, #0x80
.text:0000006C14667930 57 D0 3B D5             MRS             X23, #3, c13, c0, #2
.text:0000006C14667934 EC 16 40 F9             LDR             X12, [X23,#0x28]
.text:0000006C14667938 C9 04 00 F0             ADRP            X9, #byte_6C14702FA0@PAGE
.text:0000006C1466793C F5 03 02 2A             MOV             W21, W2
.text:0000006C14667940 F3 03 01 AA             MOV             X19, X1
.text:0000006C14667944 EC 1F 00 F9             STR             X12, [SP,#0x80+var_48]
.text:0000006C14667948 4C 07 00 B0             ADRP            X12, #a0123456789abcd_0@PAGE ; "0123456789abcdefghijklmnopqrstuvwxyzABC"...
    .text:0000006C1466794C F4 03 00 AA             MOV             X20, X0
.text:0000006C14667950 E8 03 1F AA             MOV             X8, XZR
.text:0000006C14667954 29 81 3E 91             ADD             X9, X9, #byte_6C14702FA0@PAGEOFF
.text:0000006C14667958 6A 19 83 52             MOV             W10, #0x18CB
.text:0000006C1466795C 6B 17 83 52             MOV             W11, #0x18BB
.text:0000006C14667960 8C 71 2E 91             ADD             X12, X12, #a0123456789abcd_0@PAGEOFF ; "0123456789abcdefghijklmnopqrstuvwxyzABC"...
    .text:0000006C14667960
.text:0000006C14667964
.text:0000006C14667964                         DecString_loc_6F65479964      ; CODE XREF: random_base64_sub_6F65479914+74↓j
.text:0000006C14667964 0E 0D 40 92             AND             X14, X8, #0xF
.text:0000006C14667968 2D 01 08 8B             ADD             X13, X9, X8
.text:0000006C1466796C 2E 01 0E 8B             ADD             X14, X9, X14
.text:0000006C14667970 AD 69 6A 38             LDRB            W13, [X13,X10]
.text:0000006C14667974 CE 69 6B 38             LDRB            W14, [X14,X11]
.text:0000006C14667978 CD 01 0D 4A             EOR             W13, W14, W13
.text:0000006C1466797C 8D 69 28 38             STRB            W13, [X12,X8] ; 0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
.text:0000006C14667980 08 05 00 91             ADD             X8, X8, #1
.text:0000006C14667984 1F 5D 01 F1             CMP             X8, #0x57 ; 'W'
    .text:0000006C14667988 E1 FE FF 54             B.NE            DecString_loc_6F65479964
.text:0000006C14667988
.text:0000006C1466798C                         ;   try {
.text:0000006C1466798C 41 07 00 B0 21 70 2E 91 ADRL            X1, a0123456789abcd_0 ; s
.text:0000006C14667994 E0 63 00 91             ADD             X0, SP, #0x80+var_68 ; int
.text:0000006C14667998 E2 83 00 91             ADD             X2, SP, #0x80+var_60
.text:0000006C1466799C DE EE 01 94             BL              strlen_new_sub_77982AB514
.text:0000006C1466799C                         ;   } // starts at 6C1466798C
.text:0000006C1466799C
.text:0000006C146679A0 56 07 00 B0 D6 D2 2F 91 ADRL            X22, asc_6C14750BF4 ; "\""
    .text:0000006C146679A8 48 04 80 52             MOV             W8, #0x22 ; '"'
    .text:0000006C146679AC E0 03 16 AA             MOV             X0, X22       ; s
.text:0000006C146679B0 C8 02 00 79             STRH            W8, [X22]     ; "\""
    .text:0000006C146679B4 37 C6 FE 97             BL              .strlen
.text:0000006C146679B4
.text:0000006C146679B8 E2 03 00 AA             MOV             X2, X0
.text:0000006C146679BC                         ;   try {
.text:0000006C146679BC E0 63 00 91             ADD             X0, SP, #0x80+var_68
.text:0000006C146679C0 E1 03 16 AA             MOV             X1, X22
.text:0000006C146679C4 A1 F0 01 94             BL              nop_sub_6F654F5C48
.text:0000006C146679C4                         ;   } // starts at 6C146679BC
.text:0000006C146679C4
.text:0000006C146679C8 41 07 00 B0             ADRP            X1, #dword_6C14750BF8@PAGE
.text:0000006C146679CC 88 C5 85 52             MOV             W8, #0x2E2C
.text:0000006C146679D0 21 E0 2F 91             ADD             X1, X1, #dword_6C14750BF8@PAGEOFF ; s
.text:0000006C146679D4 88 C7 A7 72             MOVK            W8, #0x3E3C,LSL#16
.text:0000006C146679D8 E9 E5 87 52             MOV             W9, #0x3F2F
.text:0000006C146679DC 28 00 00 B9             STR             W8, [X1]
.text:0000006C146679E0 29 08 00 79             STRH            W9, [X1,#(word_6C14750BFC - 0x6C14750BF8)]
.text:0000006C146679E4 3F 18 00 39             STRB            WZR, [X1,#(byte_6C14750BFE - 0x6C14750BF8)]
.text:0000006C146679E8                         ;   try {
.text:0000006C146679E8 E0 43 00 91             ADD             X0, SP, #0x80+var_70 ; int
.text:0000006C146679EC E2 83 00 91             ADD             X2, SP, #0x80+var_60
.text:0000006C146679F0 C9 EE 01 94             BL              strlen_new_sub_77982AB514
.text:0000006C146679F0                         ;   } // starts at 6C146679E8
.text:0000006C146679F0
.text:0000006C146679F4                         ;   try {
.text:0000006C146679F4 E0 23 00 91             ADD             X0, SP, #0x80+var_78
.text:0000006C146679F8 E1 63 00 91             ADD             X1, SP, #0x80+var_68
.text:0000006C146679FC FC F3 01 94             BL              nop_sub_779C0649EC
.text:0000006C146679FC                         ;   } // starts at 6C146679F4
.text:0000006C146679FC
.text:0000006C14667A00                         ;   try {
.text:0000006C14667A00 E0 23 00 91             ADD             X0, SP, #0x80+var_78
.text:0000006C14667A04 E1 43 00 91             ADD             X1, SP, #0x80+var_70
.text:0000006C14667A08 60 F0 01 94             BL              new_sub_779C063B88
.text:0000006C14667A08                         ;   } // starts at 6C14667A00
.text:0000006C14667A08
.text:0000006C14667A0C E8 07 40 F9             LDR             X8, [SP,#0x80+var_78]
.text:0000006C14667A10 18 81 5E B8             LDUR            W24, [X8,#-0x18]
.text:0000006C14667A14 FF 7F 02 A9             STP             XZR, XZR, [SP,#0x80+var_60]
.text:0000006C14667A18 FF C3 00 39             STRB            WZR, [SP,#0x80+var_50]
.text:0000006C14667A1C 88 22 40 F9             LDR             X8, [X20,#0x40]
.text:0000006C14667A20 16 A1 40 F9             LDR             X22, [X8,#0x140]
.text:0000006C14667A24 08 AD 40 F9             LDR             X8, [X8,#0x158]
.text:0000006C14667A28                         ;   try {
.text:0000006C14667A28 E0 03 1F AA             MOV             X0, XZR
.text:0000006C14667A2C 00 01 3F D6             BLR             X8            ; time
.text:0000006C14667A2C
.text:0000006C14667A30 C0 02 3F D6             BLR             X22           ; srand
.text:0000006C14667A30                         ;   } // starts at 6C14667A28
.text:0000006C14667A30
.text:0000006C14667A34 88 22 40 F9             LDR             X8, [X20,#0x40]
.text:0000006C14667A38 BF 06 00 71             CMP             W21, #1
.text:0000006C14667A3C B5 7E 40 93             SXTW            X21, W21
.text:0000006C14667A40 AB 02 00 54             B.LT            loc_6C14667A94
.text:0000006C14667A40
.text:0000006C14667A44 F9 03 1F AA             MOV             X25, XZR
.text:0000006C14667A48 FA 83 00 91             ADD             X26, SP, #0x80+var_60
.text:0000006C14667A48
.text:0000006C14667A4C
.text:0000006C14667A4C                         loc_6C14667A4C                ; CODE XREF: random_base64_sub_6F65479914+17C↓j
.text:0000006C14667A4C 08 A5 40 F9             LDR             X8, [X8,#0x148]
.text:0000006C14667A50                         ;   try {
.text:0000006C14667A50 00 01 3F D6             BLR             X8            ; rand
.text:0000006C14667A50
.text:0000006C14667A54 E8 07 40 F9             LDR             X8, [SP,#0x80+var_78]
.text:0000006C14667A58 F6 03 00 2A             MOV             W22, W0
.text:0000006C14667A5C 09 81 5F B8             LDUR            W9, [X8,#-8]
.text:0000006C14667A60 89 00 F8 37             TBNZ            W9, #0x1F, copy_random_loc_6C14667A70
.text:0000006C14667A60
.text:0000006C14667A64 E0 23 00 91             ADD             X0, SP, #0x80+var_78
.text:0000006C14667A68 4C F2 01 94             BL              memmove_sub_779C064398
.text:0000006C14667A68                         ;   } // starts at 6C14667A50
.text:0000006C14667A68
.text:0000006C14667A6C E8 07 40 F9             LDR             X8, [SP,#0x80+var_78]
.text:0000006C14667A6C
.text:0000006C14667A70
.text:0000006C14667A70                         copy_random_loc_6C14667A70    ; CODE XREF: random_base64_sub_6F65479914+14C↑j
.text:0000006C14667A70 C9 0E D8 1A             SDIV            W9, W22, W24
.text:0000006C14667A74 29 D9 18 1B             MSUB            W9, W9, W24, W22
.text:0000006C14667A78 29 7D 40 93             SXTW            X9, W9
.text:0000006C14667A7C 08 69 69 38             LDRB            W8, [X8,X9]
.text:0000006C14667A80 48 6B 39 38             STRB            W8, [X26,X25]
.text:0000006C14667A84 88 22 40 F9             LDR             X8, [X20,#0x40]
.text:0000006C14667A88 39 07 00 91             ADD             X25, X25, #1
.text:0000006C14667A8C 3F 03 15 EB             CMP             X25, X21
.text:0000006C14667A90 EB FD FF 54             B.LT            loc_6C14667A4C
.text:0000006C14667A90
.text:0000006C14667A94
.text:0000006C14667A94                         loc_6C14667A94                ; CODE XREF: random_base64_sub_6F65479914+12C↑j
.text:0000006C14667A94 08 A9 40 F9             LDR             X8, [X8,#0x150]
.text:0000006C14667A98                         ;   try {
.text:0000006C14667A98 E1 83 00 91             ADD             X1, SP, #0x80+var_60
.text:0000006C14667A9C E0 03 13 AA             MOV             X0, X19
.text:0000006C14667AA0 E2 03 15 AA             MOV             X2, X21
.text:0000006C14667AA4 00 01 3F D6             BLR             X8            ; memcpy
.text:0000006C14667AA4                         ;   } // starts at 6C14667A98
.text:0000006C14667AA4
.text:0000006C14667AA8 E8 07 40 F9             LDR             X8, [SP,#0x80+var_78]
.text:0000006C14667AAC 33 07 00 D0             ADRP            X19, #off_6C1474DF48@PAGE
.text:0000006C14667AB0 73 A6 47 F9             LDR             X19, [X19,#off_6C1474DF48@PAGEOFF]
.text:0000006C14667AB4 00 61 00 D1             SUB             X0, X8, #0x18 ; ptr
.text:0000006C14667AB8 1F 00 13 EB             CMP             X0, X19
.text:0000006C14667ABC 81 02 00 54             B.NE            loc_6C14667B0C
.text:0000006C14667ABC
.text:0000006C14667AC0
.text:0000006C14667AC0                         loc_6C14667AC0                ; CODE XREF: random_base64_sub_6F65479914+274↓j
.text:0000006C14667AC0                                                       ; random_base64_sub_6F65479914+280↓j
.text:0000006C14667AC0 E8 0B 40 F9             LDR             X8, [SP,#0x80+var_70]
.text:0000006C14667AC4 00 61 00 D1             SUB             X0, X8, #0x18 ; ptr
.text:0000006C14667AC8 1F 00 13 EB             CMP             X0, X19
.text:0000006C14667ACC 21 03 00 54             B.NE            loc_6C14667B30
.text:0000006C14667ACC
.text:0000006C14667AD0
.text:0000006C14667AD0                         loc_6C14667AD0                ; CODE XREF: random_base64_sub_6F65479914+294↓j
.text:0000006C14667AD0                                                       ; random_base64_sub_6F65479914+2A0↓j
.text:0000006C14667AD0 E8 0F 40 F9             LDR             X8, [SP,#0x80+var_68]
.text:0000006C14667AD4 00 61 00 D1             SUB             X0, X8, #0x18 ; ptr
.text:0000006C14667AD8 1F 00 13 EB             CMP             X0, X19
.text:0000006C14667ADC C1 03 00 54             B.NE            loc_6C14667B54
.text:0000006C14667ADC
.text:0000006C14667AE0
.text:0000006C14667AE0                         loc_6C14667AE0                ; CODE XREF: random_base64_sub_6F65479914+2B4↓j
.text:0000006C14667AE0                                                       ; random_base64_sub_6F65479914+2C0↓j
.text:0000006C14667AE0 E8 16 40 F9             LDR             X8, [X23,#0x28]
.text:0000006C14667AE4 E9 1F 40 F9             LDR             X9, [SP,#0x80+var_48]
.text:0000006C14667AE8 1F 01 09 EB             CMP             X8, X9
.text:0000006C14667AEC 61 07 00 54             B.NE            loc_6C14667BD8
.text:0000006C14667AEC
.text:0000006C14667AF0 FD 7B 48 A9             LDP             X29, X30, [SP,#0x80+var_s0]
.text:0000006C14667AF4 F4 4F 47 A9             LDP             X20, X19, [SP,#0x80+var_10]
.text:0000006C14667AF8 F6 57 46 A9             LDP             X22, X21, [SP,#0x80+var_20]
.text:0000006C14667AFC F8 5F 45 A9             LDP             X24, X23, [SP,#0x80+var_30]
.text:0000006C14667B00 FA 67 44 A9             LDP             X26, X25, [SP,#0x80+var_40]
.text:0000006C14667B04 FF 43 02 91             ADD             SP, SP, #0x90
.text:0000006C14667B08 C0 03 5F D6             RET
```

**5.2、加密AES KEY**

RSA公钥加密AES KEY

```makefile
.text:0000006C1466A300 FF 43 03 D1             SUB             SP, SP, #0xD0
.text:0000006C1466A304 F9 43 00 F9             STR             X25, [SP,#0xC0+var_40]
.text:0000006C1466A308 F8 5F 09 A9             STP             X24, X23, [SP,#0xC0+var_30]
.text:0000006C1466A30C F6 57 0A A9             STP             X22, X21, [SP,#0xC0+var_20]
.text:0000006C1466A310 F4 4F 0B A9             STP             X20, X19, [SP,#0xC0+var_10]
.text:0000006C1466A314 FD 7B 0C A9             STP             X29, X30, [SP,#0xC0+var_s0]
.text:0000006C1466A318 FD 03 03 91             ADD             X29, SP, #0xC0
.text:0000006C1466A31C 59 D0 3B D5             MRS             X25, #3, c13, c0, #2
.text:0000006C1466A320 28 17 40 F9             LDR             X8, [X25,#0x28]
.text:0000006C1466A324 F4 03 00 AA             MOV             X20, X0
.text:0000006C1466A328 E0 03 1F 2A             MOV             W0, WZR
.text:0000006C1466A32C 5F 40 00 71             CMP             W2, #0x10
.text:0000006C1466A330 A8 83 1B F8             STUR            X8, [X29,#var_48]
.text:0000006C1466A334 E1 09 00 54             B.NE            loc_6C1466A470
.text:0000006C1466A334
.text:0000006C1466A338 F6 03 01 AA             MOV             X22, X1
.text:0000006C1466A33C A1 09 00 B4             CBZ             X1, loc_6C1466A470
.text:0000006C1466A33C
.text:0000006C1466A340 F3 03 03 AA             MOV             X19, X3
.text:0000006C1466A344 63 09 00 B4             CBZ             X3, loc_6C1466A470
.text:0000006C1466A344
.text:0000006C1466A348 18 07 00 F0             ADRP            X24, #off_6C1474DF48@PAGE
.text:0000006C1466A34C 18 A7 47 F9             LDR             X24, [X24,#off_6C1474DF48@PAGEOFF]
.text:0000006C1466A350 BF 30 00 71             CMP             W5, #0xC
.text:0000006C1466A354 09 63 00 91             ADD             X9, X24, #0x18
.text:0000006C1466A358 A9 83 1A F8             STUR            X9, [X29,#var_58]
.text:0000006C1466A35C 28 08 00 54             B.HI            loc_6C1466A460
.text:0000006C1466A35C
.text:0000006C1466A360 88 02 40 F9             LDR             X8, [X20]
.text:0000006C1466A364 F5 03 04 AA             MOV             X21, X4
.text:0000006C1466A368 09 21 40 F9             LDR             X9, [X8,#0x40]
.text:0000006C1466A36C                         ;   try {
.text:0000006C1466A36C E8 83 00 91             ADD             X8, SP, #0xC0+var_A0
.text:0000006C1466A370 E0 03 14 AA             MOV             X0, X20
.text:0000006C1466A374 E1 03 05 2A             MOV             W1, W5
.text:0000006C1466A378 20 01 3F D6             BLR             X9            ; 解密rsa key
.text:0000006C1466A378                         ;   } // starts at 6C1466A36C
.text:0000006C1466A378
.text:0000006C1466A37C                         ;   try {
.text:0000006C1466A37C A0 63 01 D1             SUB             X0, X29, #-var_58
.text:0000006C1466A380 E1 83 00 91             ADD             X1, SP, #0xC0+var_A0
.text:0000006C1466A384 8F E6 01 94             BL              nop_sub_6D5294CDC0
.text:0000006C1466A384                         ;   } // starts at 6C1466A37C
.text:0000006C1466A384
.text:0000006C1466A388 E8 13 40 F9             LDR             X8, [SP,#0xC0+var_A0]
.text:0000006C1466A38C 00 61 00 D1             SUB             X0, X8, #0x18 ; ptr
.text:0000006C1466A390 1F 00 18 EB             CMP             X0, X24
.text:0000006C1466A394 01 0A 00 54             B.NE            loc_6C1466A4D4
.text:0000006C1466A394
.text:0000006C1466A398
.text:0000006C1466A398                         loc_6C1466A398                ; CODE XREF: RSA_Enc_AESKEY_sub_6F7389D300+234↓j
.text:0000006C1466A398                                                       ; RSA_Enc_AESKEY_sub_6F7389D300+240↓j
.text:0000006C1466A398 00 07 00 F0             ADRP            X0, #off_6C1474DD68@PAGE
.text:0000006C1466A39C 01 07 00 F0             ADRP            X1, #off_6C1474DE58@PAGE
.text:0000006C1466A3A0 00 B4 46 F9             LDR             X0, [X0,#off_6C1474DD68@PAGEOFF] ; dest
.text:0000006C1466A3A4 21 2C 47 F9             LDR             X1, [X1,#off_6C1474DE58@PAGEOFF] ; src
.text:0000006C1466A3A8 02 35 80 52             MOV             W2, #0x1A8    ; n
.text:0000006C1466A3AC 7D BA FE 97             BL              .memcpy
.text:0000006C1466A3AC
.text:0000006C1466A3B0                         ;   try {
.text:0000006C1466A3B0 00 07 00 F0             ADRP            X0, #off_6C1474DED8@PAGE
.text:0000006C1466A3B4 00 6C 47 F9             LDR             X0, [X0,#off_6C1474DED8@PAGEOFF] ; void *
.text:0000006C1466A3B8 E6 19 01 94             BL              memcmp_sub_6F738E3B50
.text:0000006C1466A3B8
.text:0000006C1466A3BC 1F 04 00 31             CMN             W0, #1
.text:0000006C1466A3C0 A0 04 00 54             B.EQ            loc_6C1466A454
.text:0000006C1466A3C0
.text:0000006C1466A3C4 A0 83 5A F8             LDUR            X0, [X29,#var_58]
.text:0000006C1466A3C8 01 80 5E F8             LDUR            X1, [X0,#-0x18]
.text:0000006C1466A3CC E2 83 00 91             ADD             X2, SP, #0xC0+var_A0
.text:0000006C1466A3D0 F5 24 01 94             BL              mp_cmp_d_sub_6F738E67A4 ; 初始化rsa pub
.text:0000006C1466A3D0
.text:0000006C1466A3D4 00 04 00 35             CBNZ            W0, loc_6C1466A454
.text:0000006C1466A3D4
.text:0000006C1466A3D8 E8 03 16 32             MOV             W8, #0x400
.text:0000006C1466A3DC A8 02 00 B9             STR             W8, [X21]
.text:0000006C1466A3E0 88 06 40 F9             LDR             X8, [X20,#8]
.text:0000006C1466A3E4 08 99 40 F9             LDR             X8, [X8,#0x130]
.text:0000006C1466A3E8 E0 03 16 32             MOV             W0, #0x400
.text:0000006C1466A3EC 00 01 3F D6             BLR             X8
.text:0000006C1466A3EC
.text:0000006C1466A3F0 88 06 40 F9             LDR             X8, [X20,#8]
.text:0000006C1466A3F4 A2 02 80 B9             LDRSW           X2, [X21]
.text:0000006C1466A3F8 F7 03 00 AA             MOV             X23, X0
.text:0000006C1466A3FC 08 91 40 F9             LDR             X8, [X8,#0x120]
.text:0000006C1466A400 E1 03 1F 2A             MOV             W1, WZR
.text:0000006C1466A404 00 01 3F D6             BLR             X8
.text:0000006C1466A404
.text:0000006C1466A408 E8 83 00 91             ADD             X8, SP, #0xC0+var_A0
.text:0000006C1466A40C E9 03 00 32             MOV             W9, #1
.text:0000006C1466A410 E1 03 1C 32             MOV             W1, #0x10
.text:0000006C1466A414 E0 03 16 AA             MOV             X0, X22
.text:0000006C1466A418 E2 03 17 AA             MOV             X2, X23
.text:0000006C1466A41C E3 03 15 AA             MOV             X3, X21
.text:0000006C1466A420 E4 03 1F AA             MOV             X4, XZR
.text:0000006C1466A424 E5 03 1F AA             MOV             X5, XZR
.text:0000006C1466A428 E6 03 1F AA             MOV             X6, XZR
.text:0000006C1466A42C E7 03 1F 2A             MOV             W7, WZR
.text:0000006C1466A430 E8 0B 00 F9             STR             X8, [SP,#0xC0+var_B0]
.text:0000006C1466A434 E9 0B 00 B9             STR             W9, [SP,#0xC0+var_B8]
.text:0000006C1466A438 FF 03 00 B9             STR             WZR, [SP,#0xC0+var_C0]
.text:0000006C1466A43C C6 23 01 94             BL              RSA_ENC_KEY_sub_6F738E6354
.text:0000006C1466A43C
.text:0000006C1466A440 E0 02 00 34             CBZ             W0, loc_6C1466A49C
.text:0000006C1466A440
.text:0000006C1466A444 88 06 40 F9             LDR             X8, [X20,#8]
.text:0000006C1466A448 08 9D 40 F9             LDR             X8, [X8,#0x138]
.text:0000006C1466A44C E0 03 17 AA             MOV             X0, X23
.text:0000006C1466A450 00 01 3F D6             BLR             X8
.text:0000006C1466A450
.text:0000006C1466A454
.text:0000006C1466A454                         loc_6C1466A454                ; CODE XREF: RSA_Enc_AESKEY_sub_6F7389D300+C0↑j
.text:0000006C1466A454                                                       ; RSA_Enc_AESKEY_sub_6F7389D300+D4↑j
.text:0000006C1466A454 E0 03 1F 2A             MOV             W0, WZR
.text:0000006C1466A454
.text:0000006C1466A458
.text:0000006C1466A458                         loc_6C1466A458                ; CODE XREF: RSA_Enc_AESKEY_sub_6F7389D300+1AC↓j
.text:0000006C1466A458 A9 83 5A F8             LDUR            X9, [X29,#var_58]
.text:0000006C1466A45C 02 00 00 14             B               loc_6C1466A464
.text:0000006C1466A45C
.text:0000006C1466A460                         ; ---------------------------------------------------------------------------
.text:0000006C1466A460
.text:0000006C1466A460                         loc_6C1466A460                ; CODE XREF: RSA_Enc_AESKEY_sub_6F7389D300+5C↑j
.text:0000006C1466A460 E0 03 1F 2A             MOV             W0, WZR
.text:0000006C1466A460
.text:0000006C1466A464
.text:0000006C1466A464                         loc_6C1466A464                ; CODE XREF: RSA_Enc_AESKEY_sub_6F7389D300+15C↑j
.text:0000006C1466A464 28 61 00 D1             SUB             X8, X9, #0x18
.text:0000006C1466A468 1F 01 18 EB             CMP             X8, X24
.text:0000006C1466A46C 21 02 00 54             B.NE            loc_6C1466A4B0
.text:0000006C1466A46C
.text:0000006C1466A470
.text:0000006C1466A470                         loc_6C1466A470                ; CODE XREF: RSA_Enc_AESKEY_sub_6F7389D300+34↑j
.text:0000006C1466A470                                                       ; RSA_Enc_AESKEY_sub_6F7389D300+3C↑j
.text:0000006C1466A470                                                       ; RSA_Enc_AESKEY_sub_6F7389D300+44↑j
.text:0000006C1466A470                                                       ; RSA_Enc_AESKEY_sub_6F7389D300+208↓j
.text:0000006C1466A470                                                       ; RSA_Enc_AESKEY_sub_6F7389D300+220↓j
.text:0000006C1466A470 28 17 40 F9             LDR             X8, [X25,#0x28]
.text:0000006C1466A474 A9 83 5B F8             LDUR            X9, [X29,#var_48]
.text:0000006C1466A478 1F 01 09 EB             CMP             X8, X9
.text:0000006C1466A47C 41 06 00 54             B.NE            loc_6C1466A544
.text:0000006C1466A47C
.text:0000006C1466A480 FD 7B 4C A9             LDP             X29, X30, [SP,#0xC0+var_s0]
.text:0000006C1466A484 F4 4F 4B A9             LDP             X20, X19, [SP,#0xC0+var_10]
.text:0000006C1466A488 F6 57 4A A9             LDP             X22, X21, [SP,#0xC0+var_20]
.text:0000006C1466A48C F8 5F 49 A9             LDP             X24, X23, [SP,#0xC0+var_30]
.text:0000006C1466A490 F9 43 40 F9             LDR             X25, [SP,#0xC0+var_40]
.text:0000006C1466A494 FF 43 03 91             ADD             SP, SP, #0xD0
.text:0000006C1466A498 C0 03 5F D6             RET

公钥
30818902818100B4BD599215C5B62E4EFD19ECF5380F7B3C16FE805E2FCC6CA88D3827805F7EA910C608985A78019234231DD649D89A6EBF0D6F0980B4CF4554BB759B2F1CE7DBE30750ADECA5626BD02491E12070803A7CAC74F5A58053AB202533A5E16FDF82FD2E26699992A25DEE91B8DF9E4AED7120ADB3AEFC2D25F956FC001F753FFEB30203010001
```

**5.3、bae64加密RSA加密后AES KEY**

```makefile
.text:0000006C1466A60C F8 5F BC A9             STP             X24, X23, [SP,#-0x10+var_30]!
.text:0000006C1466A610 F6 57 01 A9             STP             X22, X21, [SP,#0x30+var_20]
.text:0000006C1466A614 F4 4F 02 A9             STP             X20, X19, [SP,#0x30+var_10]
.text:0000006C1466A618 FD 7B 03 A9             STP             X29, X30, [SP,#0x30+var_s0]
.text:0000006C1466A61C FD C3 00 91             ADD             X29, SP, #0x30
.text:0000006C1466A620 F4 03 00 AA             MOV             X20, X0
.text:0000006C1466A624 5F 04 00 71             CMP             W2, #1
.text:0000006C1466A628 E0 03 1F 2A             MOV             W0, WZR
.text:0000006C1466A62C 4B 05 00 54             B.LT            loc_6C1466A6D4
.text:0000006C1466A62C
.text:0000006C1466A630 F6 03 01 AA             MOV             X22, X1
.text:0000006C1466A634 01 05 00 B4             CBZ             X1, loc_6C1466A6D4
.text:0000006C1466A634
.text:0000006C1466A638 F3 03 03 AA             MOV             X19, X3
.text:0000006C1466A63C C3 04 00 B4             CBZ             X3, loc_6C1466A6D4
.text:0000006C1466A63C
.text:0000006C1466A640 C9 AA 8A 52             MOV             W9, #0x5556
.text:0000006C1466A644 48 08 00 11             ADD             W8, W2, #2
.text:0000006C1466A648 A9 AA AA 72             MOVK            W9, #0x5555,LSL#16
.text:0000006C1466A64C 08 7D 29 9B             SMULL           X8, W8, W9
.text:0000006C1466A650 09 FD 7F D3             LSR             X9, X8, #0x3F ; '?'
.text:0000006C1466A654 08 FD 60 D3             LSR             X8, X8, #0x20 ; ' '
.text:0000006C1466A658 08 01 09 0B             ADD             W8, W8, W9
.text:0000006C1466A65C E9 03 00 32             MOV             W9, #1
.text:0000006C1466A660 09 75 1E 33             BFI             W9, W8, #2, #0x1E
.text:0000006C1466A664 89 00 00 B9             STR             W9, [X4]
.text:0000006C1466A668 88 06 40 F9             LDR             X8, [X20,#8]
.text:0000006C1466A66C 20 7D 40 93             SXTW            X0, W9
.text:0000006C1466A670 F5 03 04 AA             MOV             X21, X4
.text:0000006C1466A674 F7 03 02 2A             MOV             W23, W2
.text:0000006C1466A678 08 99 40 F9             LDR             X8, [X8,#0x130]
.text:0000006C1466A67C 00 01 3F D6             BLR             X8            ; malloc
.text:0000006C1466A67C
.text:0000006C1466A680 A0 02 00 B4             CBZ             X0, loc_6C1466A6D4
.text:0000006C1466A680
.text:0000006C1466A684 88 06 40 F9             LDR             X8, [X20,#8]
.text:0000006C1466A688 A2 02 80 B9             LDRSW           X2, [X21]
.text:0000006C1466A68C E1 03 1F 2A             MOV             W1, WZR
.text:0000006C1466A690 F8 03 00 AA             MOV             X24, X0
.text:0000006C1466A694 08 91 40 F9             LDR             X8, [X8,#0x120]
.text:0000006C1466A698 00 01 3F D6             BLR             X8
.text:0000006C1466A698
.text:0000006C1466A69C E1 7E 40 93             SXTW            X1, W23
.text:0000006C1466A6A0 E0 03 16 AA             MOV             X0, X22
.text:0000006C1466A6A4 E2 03 18 AA             MOV             X2, X24
.text:0000006C1466A6A8 E3 03 15 AA             MOV             X3, X21
.text:0000006C1466A6AC 93 18 01 94             BL              BASE64_sub_6F738E38F8
.text:0000006C1466A6AC
.text:0000006C1466A6B0 E0 00 00 34             CBZ             W0, loc_6C1466A6CC
.text:0000006C1466A6B0
.text:0000006C1466A6B4 88 06 40 F9             LDR             X8, [X20,#8]
.text:0000006C1466A6B8 E0 03 18 AA             MOV             X0, X24
.text:0000006C1466A6BC 08 9D 40 F9             LDR             X8, [X8,#0x138]
.text:0000006C1466A6C0 00 01 3F D6             BLR             X8
.text:0000006C1466A6C0
.text:0000006C1466A6C4 E0 03 1F 2A             MOV             W0, WZR
.text:0000006C1466A6C8 03 00 00 14             B               loc_6C1466A6D4
.text:0000006C1466A6C8
.text:0000006C1466A6CC
.text:0000006C1466A6CC                         loc_6C1466A6CC                ; CODE XREF: base64_sub_6F7389D60C+A4↑j
.text:0000006C1466A6CC 78 02 00 F9             STR             X24, [X19]
.text:0000006C1466A6D0 E0 03 00 32             MOV             W0, #1
.text:0000006C1466A6D0
.text:0000006C1466A6D4
.text:0000006C1466A6D4                         loc_6C1466A6D4                ; CODE XREF: base64_sub_6F7389D60C+20↑j
.text:0000006C1466A6D4                                                       ; base64_sub_6F7389D60C+28↑j
.text:0000006C1466A6D4                                                       ; base64_sub_6F7389D60C+30↑j
.text:0000006C1466A6D4                                                       ; base64_sub_6F7389D60C+74↑j
.text:0000006C1466A6D4                                                       ; base64_sub_6F7389D60C+BC↑j
.text:0000006C1466A6D4 FD 7B 43 A9             LDP             X29, X30, [SP,#0x30+var_s0]
.text:0000006C1466A6D8 F4 4F 42 A9             LDP             X20, X19, [SP,#0x30+var_10]
.text:0000006C1466A6DC F6 57 41 A9             LDP             X22, X21, [SP,#0x30+var_20]
.text:0000006C1466A6E0 F8 5F C4 A8             LDP             X24, X23, [SP+0x30+var_30],#0x40
.text:0000006C1466A6E4 C0 03 5F D6             RET
```

### **六、协议还原**

**6.1、AES还原**

标准AES算法

```java
    








    public static byte[] encrypt(String content, String password) {
        try {
            KeyGenerator kgen = KeyGenerator.getInstance("AES");
            kgen.init(128, new SecureRandom(password.getBytes()));
            SecretKey secretKey = kgen.generateKey();
            byte[] enCodeFormat = secretKey.getEncoded();
            SecretKeySpec key = new SecretKeySpec(enCodeFormat, "AES");
            Cipher cipher = Cipher.getInstance("AES");
            byte[] byteContent = content.getBytes("utf-8");
            cipher.init(Cipher.ENCRYPT_MODE, key);
            byte[] result = cipher.doFinal(byteContent);
            return result;
        } catch (NoSuchPaddingException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        } catch (IllegalBlockSizeException e) {
            e.printStackTrace();
        } catch (BadPaddingException e) {
            e.printStackTrace();
        }
        return null;
    }
```

**6.2、RSA公钥转换**

从内存中提取出来的RSA公钥是pkcs#1格式需要转换成pkcs#8格式方便java使用。

```cs
    public static byte[] formatPublicKeyPKCS1ToPKCS8(byte[] pkcs1PublicKeyByte) {

        RSAPublicKey rsaPub = RSAPublicKey.getInstance(pkcs1PublicKeyByte);

        byte[] pkcs8Bytes = null;
        try {
            KeyFactory kf = KeyFactory.getInstance("RSA");

            PublicKey generatePublic = kf.generatePublic(new RSAPublicKeySpec(rsaPub.getModulus(), rsaPub.getPublicExponent()));

            pkcs8Bytes = generatePublic.getEncoded();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (InvalidKeySpecException e) {
            e.printStackTrace();
        }

        return pkcs8Bytes;

    }

        byte[] messageEncodeBytes = new byte[2048];
        
        byte[] RsaPubbuffer = new byte[]{
                0x30, (byte) 0x81, (byte) 0x89, 0x02, (byte) 0x81, (byte) 0x81, 0x00, (byte) 0xB4, (byte) 0xBD, 0x59, (byte) 0x92, 0x15, (byte) 0xC5, (byte) 0xB6, 0x2E, 0x4E,
                (byte) 0xFD, 0x19, (byte) 0xEC, (byte) 0xF5, 0x38, 0x0F, 0x7B, 0x3C, 0x16, (byte) 0xFE, (byte) 0x80, 0x5E, 0x2F, (byte) 0xCC, 0x6C, (byte) 0xA8,
                (byte) 0x8D, 0x38, 0x27, (byte) 0x80, 0x5F, 0x7E, (byte) 0xA9, 0x10, (byte) 0xC6, 0x08, (byte) 0x98, 0x5A, 0x78, 0x01, (byte) 0x92, 0x34,
                0x23, 0x1D, (byte) 0xD6, 0x49, (byte) 0xD8, (byte) 0x9A, 0x6E, (byte) 0xBF, 0x0D, 0x6F, 0x09, (byte) 0x80, (byte) 0xB4, (byte) 0xCF, 0x45, 0x54,
                (byte) 0xBB, 0x75, (byte) 0x9B, 0x2F, 0x1C, (byte) 0xE7, (byte) 0xDB, (byte) 0xE3, 0x07, 0x50, (byte) 0xAD, (byte) 0xEC, (byte) 0xA5, 0x62, 0x6B, (byte) 0xD0,
                0x24, (byte) 0x91, (byte) 0xE1, 0x20, 0x70, (byte) 0x80, 0x3A, 0x7C, (byte) 0xAC, 0x74, (byte) 0xF5, (byte) 0xA5, (byte) 0x80, 0x53, (byte) 0xAB, 0x20,
                0x25, 0x33, (byte) 0xA5, (byte) 0xE1, 0x6F, (byte) 0xDF, (byte) 0x82, (byte) 0xFD, 0x2E, 0x26, 0x69, (byte) 0x99, (byte) 0x92, (byte) 0xA2, 0x5D, (byte) 0xEE,
                (byte) 0x91, (byte) 0xB8, (byte) 0xDF, (byte) 0x9E, 0x4A, (byte) 0xED, 0x71, 0x20, (byte) 0xAD, (byte) 0xB3, (byte) 0xAE, (byte) 0xFC, 0x2D, 0x25, (byte) 0xF9, 0x56,
                (byte) 0xFC, 0x00, 0x1F, 0x75, 0x3F, (byte) 0xFE, (byte) 0xB3, 0x02, 0x03, 0x01, 0x00, 0x01};

        byte[] pkcs8 = RsaPkcsTransformer.formatPublicKeyPKCS1ToPKCS8(RsaPubbuffer);
```

**6.3、RSA加加密AES AEY**

```php








public static byte[] encrypt(RSAPublicKey publicKey, byte[] plainTextData) throws Exception {
    if (publicKey == null) {
        throw new Exception("加密公钥为空, 请设置");
    }
    Cipher cipher = null;
    try {
        
        cipher = Cipher.getInstance("RSA");
        cipher.init(Cipher.ENCRYPT_MODE, publicKey);
        byte[] output = cipher.doFinal(plainTextData);
        return output;
    } catch (NoSuchAlgorithmException e) {
        throw new Exception("无此加密算法");
    } catch (NoSuchPaddingException e) {
        e.printStackTrace();
        return null;
    } catch (InvalidKeyException e) {
        throw new Exception("加密公钥非法,请检查");
    } catch (IllegalBlockSizeException e) {
        throw new Exception("明文长度非法");
    } catch (BadPaddingException e) {
        throw new Exception("明文数据已损坏");
    }
}

        try {
            RSAPublicKey Pubpcks1 = (RSAPublicKey) RsaPkcsTransformer.formatPKCS8PublicKey(pkcs8);
            messageEncodeBytes = RSAUtils.encrypt( Pubpcks1, aeskey.getBytes());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return messageEncodeBytes;
最后再base64加密
```

**6.4、加密采集的设备数据**

AES加密后base64加加密

```typescript
    
    public static String EncDeviceinfo(String deviceinfo, String key){
        String ret  = "";
        byte[] encdeviceinfo  = AESUtil.encrypt(deviceinfo, key);
        ret = Base64.encode(encdeviceinfo);
        System.out.println("\r\nEncDeviceinfo:\n"+ret);
        return ret;
    }
```

**6.5、发送服务器返回设备指纹**

将加密后的设备信息组合发送给服务器，服务器返回设备指纹。

```cs
    public static String doPost(String serurl, byte[] body) throws MalformedURLException, HttpException {
        int code = 0;
        if (null == body || serurl.isEmpty() || serurl.equals("")) {
            return null;
        }
        Builder builder = new Builder();
        HttpUtil httpclient = builder.build(serurl);
        HttpResponse response = httpclient.post(body);
        if (null != response){
            code = response.code;
        }

        System.out.println("doRequest code :"+code);
        return response.getString();
    }

        try {
           String ret =  HttpUtil.doPost("https://ac.dun.xxxyun.com/v2/m/d", encBody.getBytes());
           System.out.print("ret: "+ret);
        } catch (MalformedURLException e) {
            e.printStackTrace();
        } catch (HttpException e) {
            e.printStackTrace();
        }
```

通过还原的协议模拟，服务器成功返回

```json
{"code":200,"msg":"ok","result":{"timestamp":1675833206592,"tid":"jBqlPTfZupNAUUUUUBOAeE37P0ZsamVQ","dt":"Zl/zPvkXgopBAEBEAReROVmuK0Y8PmQV","ni":"JyR+on5ZMtDZRV1lDWN+BiDWM3/osPKRJ4ZhDIMMuzbXVliI40BIsN7x0aETYmEZQU8\u003d"}}
```

### **七、总结**

移动端反作弊其中最重要的是能够唯一标识一台设备，是否够精确定位一个设备才能与其它设备数据、行为数据进行关联，从而判断一个设备的风险，设备指纹作为风控对抗的基础，判断其是否优质最重要指标是唯一性和稳定性。亿盾指纹从安卓11以后指纹稳定性相对较差，主要原因是几个关键的指纹生成特征未能获取到。

其中一些关键的信息采集svc 0方式获取，能防止常见的改机工具，但是深度ROM定制与更底层的改机很难防止。

样本获取方式，关注公众号，公众号输入框回复“xp” 获取下载链接。