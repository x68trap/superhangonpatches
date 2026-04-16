# スーパーハングオン for X68030/060 パッチ配布用リポジトリ

X68000用ソフト スーパーハングオン を、X68030で動作させるためのパッチファイルです。
68060 を積んだ機種の場合（68060turbo）、movep命令が使えず描画が遅い為、060用パッチも用意しています。
（変更箇所が多く、従来のパッチツールでは対応できなかった為、bsdiff/bspatch でパッチを生成しています）

以下の改修を施しています。
- X68030 以上への対応
- movep 命令を使わない 060用パッチも別に用意
- X68000 XVI 16MHz 等の高クロック機で音がおかしくなる問題への対策
- 『電脳倶楽部 1990年3月号（Vol.22）』掲載『スーパーハングオン』巨大化プログラム(石本淳様)パッチ内容取り込み

改修により、デカキャラパッチが当たらなくなってしまった為、パッチ内容を取り込ませていただきました。
デカキャラパッチのソースを公開されていた石本様に深く感謝します。

## ダウンロード
最新の実行ファイルは、以下のリリースページからダウンロードしてください。

[👉 最新版をダウンロードする](https://github.com/x68trap/superhangonpatches/releases/latest)

***
# 改修ポイント

## M2.X
### OPMアクセス時のBUSY中待ち処理追加
元がBUSY中チェックをしていなかったので、待ちループを追加。
```
L00002c:
	move.b	d1,($00e90001)
.ifndef ORIGINAL
1:	tst.b	($00e90003)
	bmi	1b
.endif
	move.b	d0,($00e90003)
```
### MIDIボードチェックのバスエラー処理部を 68060 まで対応
68010 以降で例外時のスタックフレームが異なる為、こんな感じで対応。
```
BUSERR_68030:
; Format
;	0000	4word
;	0001	4word
;	0010	6word	アドレスエラー 68040
;	0011	6word	68060 コプロ
;	0100	8word	バスエラー 68060
;	0111	30word	バスエラー 68040
;	1001	10word	68030 コプロ
;	1010	16word	68030 アドレスエラーまたはバスエラー（ショート）
;	1011	46word	68030 アドレスエラーまたはバスエラー（ロング）

	movem.l	d0/a0-a1,-(a7)

	move.w	(6+12,a7),d0		; Format & Vector Offset
	andi.w	#$f000,d0			; Format部のみにする

	lea.l	(16+12,a7),a0		; stuck再構築アドレス
	lea.l	(8+12,a7),a1		; data fault address
	cmpi.w	#$4000,d0			; for 68060 8word stuck
	beq	Common_buserror

	lea.l	(60+12,a7),a0
	lea.l	(20+12,a7),a1		; data fault address
	cmpi.w	#$7000,d0			; for 68040 Bus Error 30word stuck
	beq	Common_buserror

	lea.l	(32+12,a7),a0
	lea.l	(16+12,a7),a1		; data fault address
	cmpi.w	#$a000,d0			; for 68030 Bus Error Short stuck (16word)
	beq	Common_buserror

	lea.l	(92+12,a7),a0
	lea.l	(16+12,a7),a1		; data fault address
	cmpi.w	#$b000,d0			; for 68030 Bus Error Long stuck (46word)
	beq	Common_buserror

Original_buserror:
	movem.l	(a7)+,d0/a0-a1

	move.l	(L001544),-(a7)		; 本来のオリジナルのバスエラー処理へ
	rts

Common_buserror:
	cmpi.l	#$00eafa05,(a1)		; MIDIボードへのアクセスだったか？
	bne.s	Original_buserror	; 違う場合は本来のバスエラー処理へ飛ばす。

	clr.w	-(a0)				; Format番号 + ベクタオフセット
	move.l	#L001062,-(a0)		; 戻りたいアドレス
	move.w	(0+12,a7),-(a0)		; SR

	move.l	(8,a7),-(a0)		; 先に保存していた a1
	move.l	(4,a7),-(a0)		; 先に保存していた a0
	move.l	(a7),-(a0)			; 先に保存していた d0

	movea.l	a0,a7				; a0 の位置が新しい sp 位置

	movem.l	(a7)+,d0/a0-a1
	rte
```

## M2MOP.X
### OPMアクセス時のBUSY中待ち処理追加
元がBUSY中チェックをしていなかったので、待ちループを追加。
```
L000060:
	tst.b	($0003,a6)
	bmi	L000060
	move.b	#$1b,($0001,a6)
.ifndef ORIGINAL
1:	tst.b	($0003,a6)
	bmi	1b
.endif
```
### OPMアクセス時のBUSY中待ち処理追加
元がBUSY中チェックをしていなかったので、待ちループを追加。
```
L00012e:
	tst.b	($0003,a1)
	bmi	L00012e
	move.b	#$1b,($0001,a1)
.ifndef ORIGINAL
1:	tst.b	($0003,a1)
	bmi	1b
.endif
	move.b	d0,($0003,a1)
```
## M2OPM.X
### OPMアクセス時のBUSY中待ち処理追加
元がBUSY中チェックをしていなかったので、待ちループを追加。
```
L000062:
	move.b	d1,($00e90001)
.ifndef ORIGINAL
1:	tst.b	($00e90003)
	bmi	1b
.endif
	move.b	d0,($00e90003)
```

## SH.X
### アナログジョイスティック用のタイムアウト値の増強
キャッシュ搭載機では単純に計算できないのだけど、元の値では不足で早く抜けてしまう為、とりあえず倍増（安易）。
```
.ifdef ORIGINAL
	move.w	#$0080,d1		; ループ用WAIT値（10MHz:283us, 16MHz:177us, 25MHz:113us）
.else
	move.w	#$00FF,d1		; ループ用WAIT値（$00FF: 10MHz:563us, 16MHz:352us, 25MHz:281us）
.endif
```

### OPMアクセス時のBUSY待ち追加
元がBUSY中チェックをしていなかったので、待ちループを追加。
```
.ifndef ORIGINAL
	tst.b	($e9a001)
.endif
1:	tst.b	($00e90003)
	bmi	1b
	move.b	d1,($00e90001)
.ifndef ORIGINAL
	tst.b	($e9a001)
1:	tst.b	($00e90003)
	bmi	1b
.endif
	move.b	d0,($00e90003)
```

### 68060用のmovep除外対応(A)
描画処理の一部。実際には、処理を多数並べて細かい細工の上で配置されている。

movep命令搭載機用 
```
mptypeA	.macro	
	move.l	(a3)+,d0
	movep.l	d0,($0001,a1)
	move.l	(a3)+,d0
	movep.l	d0,($0009,a1)
	lea.l	($0010,a1),a1
	.endm
```

movep命令非搭載機用
```
mp060A	.macro
	move.l	(a3)+,d0
	ror.l	#8,d0		; 3 0 _ 1 2
	ror.w	#8,d0		; 3 0 _ 2 1
	move.l	d0,(a1)+
	rol.l	#8,d0		; 0 2 _ 1 3
	move.l	d0,(a1)+
	move.l	(a3)+,d0
	ror.l	#8,d0		; 3 0 _ 1 2
	ror.w	#8,d0		; 3 0 _ 2 1
	move.l	d0,(a1)+
	rol.l	#8,d0		; 0 2 _ 1 3
	move.l	d0,(a1)+
	.endm
```

### 68060用のmovep除外対応(B)
描画処理の一部。実際には、処理を多数並べて細かい細工の上で配置されている。

movep命令搭載機用 
```
mptypeB	.macro
	move.l	(a3)+,d0
	add.l	d2,d0
	movep.l	d0,($0001,a1)
	move.l	(a3)+,d0
	add.l	d2,d0
	movep.l	d0,($0009,a1)
	lea.l	($0010,a1),a1
	.endm
```

movep命令非搭載機用
```
mp060B	.macro
	move.l	(a3)+,d0
	add.l	d2,d0
	ror.l	#8,d0		; 3 0 _ 1 2
	ror.w	#8,d0		; 3 0 _ 2 1
	move.l	d0,(a1)+
	rol.l	#8,d0		; 0 2 _ 1 3
	move.l	d0,(a1)+

	move.l	(a3)+,d0
	add.l	d2,d0
	ror.l	#8,d0		; 3 0 _ 1 2
	ror.w	#8,d0		; 3 0 _ 2 1
	move.l	d0,(a1)+
	rol.l	#8,d0		; 0 2 _ 1 3
	move.l	d0,(a1)+
	.endm
```
