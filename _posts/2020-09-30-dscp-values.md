---
title: DSCP / 802.1p / WMM / IP QoS Table
---

A little overview of the DSCP values and names,
and the resulting IP ToS field (assuming that
the ECN flags are both zero).

Also, this shows the resulting 802.1p Class of Service
and WMM priority, assuming a "dumb" mapping from
the highest 3 bits of DSCP.


| DSCP | name | IP TOS | 802.1p | WMM    | prio |
| ---- | ---- | ------ | ------ | ------ | ---- |
| 0    | CS0  | 0      | 0 (BE) | AC_BE  | 1    |
| 1    |      | 4      | 0      | AC_BE  | 1    |
| 2    |      | 8      | 0      | AC_BE  | 1    |
| 3    |      | 12     | 0      | AC_BE  | 1    |
| 4    |      | 16     | 0      | AC_BE  | 1    |
| 5    |      | 20     | 0      | AC_BE  | 1    |
| 6    |      | 24     | 0      | AC_BE  | 1    |
| 7    |      | 28     | 0      | AC_BE  | 1    |
| 8    | CS1  | 32     | 1 (BK) | AC_BK  | 0    |
| 9    |      | 36     | 1      | AC_BK  | 0    |
| 10   | AF11 | 40     | 1      | AC_BK  | 0    |
| 11   |      | 44     | 1      | AC_BK  | 0    |
| 12   | AF12 | 48     | 1      | AC_BK  | 0    |
| 13   |      | 52     | 1      | AC_BK  | 0    |
| 14   | AF13 | 56     | 1      | AC_BK  | 0    |
| 15   |      | 60     | 1      | AC_BK  | 0    |
| 16   | CS2  | 64     | 2 (EE) | AC_BK  | 2    |
| 17   |      | 68     | 2      | AC_BK  | 2    |
| 18   | AF21 | 72     | 2      | AC_BK  | 2    |
| 19   |      | 76     | 2      | AC_BK  | 2    |
| 20   | AF22 | 80     | 2      | AC_BK  | 2    |
| 21   |      | 84     | 2      | AC_BK  | 2    |
| 22   | AF23 | 88     | 2      | AC_BK  | 2    |
| 23   |      | 92     | 2      | AC_BK  | 2    |
| 24   | CS3  | 96     | 3 (CA) | AC_BE  | 3    |
| 25   |      | 100    | 3      | AC_BE  | 3    |
| 26   | AF31 | 104    | 3      | AC_BE  | 3    |
| 27   |      | 108    | 3      | AC_BE  | 3    |
| 28   | AF32 | 112    | 3      | AC_BE  | 3    |
| 29   |      | 116    | 3      | AC_BE  | 3    |
| 30   | AF33 | 120    | 3      | AC_BE  | 3    |
| 31   |      | 124    | 3      | AC_BE  | 3    |
| 32   | CS4  | 128    | 4 (VI) | AC_VI  | 4    |
| 33   |      | 132    | 4      | AC_VI  | 4    |
| 34   | AF41 | 136    | 4      | AC_VI  | 4    |
| 35   |      | 140    | 4      | AC_VI  | 4    |
| 36   | AF42 | 144    | 4      | AC_VI  | 4    |
| 37   |      | 148    | 4      | AC_VI  | 4    |
| 38   | AF43 | 152    | 4      | AC_VI  | 4    |
| 39   |      | 156    | 4      | AC_VI  | 4    |
| 40   | CS5  | 160    | 5 (VO) | AC_VI  | 5    |
| 41   |      | 164    | 5      | AC_VI  | 5    |
| 42   |      | 168    | 5      | AC_VI  | 5    |
| 43   |      | 172    | 5      | AC_VI  | 5    |
| 44|VOICE-ADMIT|176   | 5      | AC_VI  | 5    |
| 45   |      | 180    | 5      | AC_VI  | 5    |
| 46   | EF   | 184    | 5      | AC_VI  | 5    |
| 47   |      | 188    | 5      | AC_VI  | 5    |
| 48   | CS6  | 192    | 6 (IC) | AC_VO  | 6    |
| 49   |      | 196    | 6      | AC_VO  | 6    |
| 50   |      | 200    | 6      | AC_VO  | 6    |
| 51   |      | 204    | 6      | AC_VO  | 6    |
| 52   |      | 208    | 6      | AC_VO  | 6    |
| 53   |      | 212    | 6      | AC_VO  | 6    |
| 54   |      | 216    | 6      | AC_VO  | 6    |
| 55   |      | 220    | 6      | AC_VO  | 6    |
| 56   | CS7  | 224    | 7 (NC) | AC_VO  | 7    |
| 57   |      | 228    | 7      | AC_VO  | 7    |
| 58   |      | 232    | 7      | AC_VO  | 7    |
| 59   |      | 236    | 7      | AC_VO  | 7    |
| 60   |      | 240    | 7      | AC_VO  | 7    |
| 61   |      | 244    | 7      | AC_VO  | 7    |
| 62   |      | 248    | 7      | AC_VO  | 7    |
| 63   |      | 252    | 7      | AC_VO  | 7    |
