# 嵌入式图像库 LVGL 
## 前言
- 对比 Mix-C ASM 与 ARM Cortex-M3 ASM
- ARM 平台开启 -Os 空间优化

## 选节一
**lvgl/src/lv_draw/lv_draw_arc.c**
```C
/**
 * Draw an arc. (Can draw pie too with great thickness.)
 * @param center_x the x coordinate of the center of the arc
 * @param center_y the y coordinate of the center of the arc
 * @param radius the radius of the arc
 * @param mask the arc will be drawn only in this mask
 * @param start_angle the start angle of the arc (0 deg on the bottom, 90 deg on the right)
 * @param end_angle the end angle of the arc
 * @param clip_area the arc will be drawn only in this area
 * @param dsc pointer to an initialized `lv_draw_line_dsc_t` variable
 */
void lv_draw_arc(
    lv_coord_t center_x, 
    lv_coord_t center_y, 
    uint16_t radius,
    uint16_t start_angle,
    uint16_t end_angle,
    const lv_area_t * clip_area,
    const lv_draw_line_dsc_t * dsc)
{
    if(dsc->opa <= LV_OPA_MIN) return;
    if(dsc->width == 0) return;
    if(start_angle == end_angle) return;

    lv_style_int_t width = dsc->width;
    if(width > radius) width = radius;

    lv_draw_rect_dsc_t cir_dsc;
    lv_draw_rect_dsc_init(&cir_dsc);
    cir_dsc.radius = LV_RADIUS_CIRCLE;
    cir_dsc.bg_opa = LV_OPA_TRANSP;
    cir_dsc.border_opa = dsc->opa;
    cir_dsc.border_color = dsc->color;
    cir_dsc.border_width = width;
    cir_dsc.border_blend_mode = dsc->blend_mode;

    lv_area_t area;
    area.x1 = center_x - radius;
    area.y1 = center_y - radius;
    area.x2 = center_x + radius - 1;  /*-1 because the center already belongs to the left/bottom part*/
    area.y2 = center_y + radius - 1;

    /*Draw a full ring*/
    if(start_angle + 360 == end_angle || start_angle == end_angle + 360) {
        lv_draw_rect(&area, clip_area, &cir_dsc);
        return;
    }

    if(start_angle >= 360) start_angle -= 360;
    if(end_angle >= 360) end_angle -= 360;

    lv_draw_mask_angle_param_t mask_angle_param;
    lv_draw_mask_angle_init(&mask_angle_param, center_x, center_y, start_angle, end_angle);

    int16_t mask_angle_id = lv_draw_mask_add(&mask_angle_param, NULL);

    int32_t angle_gap;
    if(end_angle > start_angle) {
        angle_gap = 360 - (end_angle - start_angle);
    }
    else {
        angle_gap = start_angle - end_angle;
    }
    if(angle_gap > SPLIT_ANGLE_GAP_LIMIT && radius > SPLIT_RADIUS_LIMIT) {
        /*Handle each quarter individually and skip which is empty*/
        quarter_draw_dsc_t q_dsc;
        q_dsc.center_x = center_x;
        q_dsc.center_y = center_y;
        q_dsc.radius = radius;
        q_dsc.start_angle = start_angle;
        q_dsc.end_angle = end_angle;
        q_dsc.start_quarter = (start_angle / 90) & 0x3;
        q_dsc.end_quarter = (end_angle / 90) & 0x3;
        q_dsc.width = width;
        q_dsc.draw_dsc =  &cir_dsc;
        q_dsc.draw_area = &area;
        q_dsc.clip_area = clip_area;

        draw_quarter_0(&q_dsc);
        draw_quarter_1(&q_dsc);
        draw_quarter_2(&q_dsc);
        draw_quarter_3(&q_dsc);
    }
    else {
        lv_draw_rect(&area, clip_area, &cir_dsc);
    }
    lv_draw_mask_remove_id(mask_angle_id);

    if(dsc->round_start || dsc->round_end) {
        cir_dsc.bg_color        = dsc->color;
        cir_dsc.bg_opa        = dsc->opa;
        cir_dsc.bg_blend_mode = dsc->blend_mode;
        cir_dsc.border_width = 0;

        lv_area_t round_area;
        if(dsc->round_start) {
            get_rounded_area(start_angle, radius, width, &round_area);
            round_area.x1 += center_x;
            round_area.x2 += center_x;
            round_area.y1 += center_y;
            round_area.y2 += center_y;

            lv_draw_rect(&round_area, clip_area, &cir_dsc);
        }

        if(dsc->round_end) {
            get_rounded_area(end_angle, radius, width, &round_area);
            round_area.x1 += center_x;
            round_area.x2 += center_x;
            round_area.y1 += center_y;
            round_area.y2 += center_y;

            lv_draw_rect(&round_area, clip_area, &cir_dsc);
        }
    }
}

```

```asm
00000000 <lv_draw_arc>:
   0:	e92d 4ff0 	stmdb	sp!, {r4, r5, r6, r7, r8, r9, sl, fp, lr}
   4:	b0c3      	sub	sp, #268	; 0x10c
   6:	9f4e      	ldr	r7, [sp, #312]	; 0x138
   8:	461c      	mov	r4, r3
   a:	7a3b      	ldrb	r3, [r7, #8]
   c:	4616      	mov	r6, r2
   e:	2b02      	cmp	r3, #2
  10:	e9cd 0103 	strd	r0, r1, [sp, #12]
  14:	f8bd 5130 	ldrh.w	r5, [sp, #304]	; 0x130
  18:	d944      	bls.n	a4 <lv_draw_arc+0xa4>
  1a:	f9b7 8002 	ldrsh.w	r8, [r7, #2]
  1e:	f1b8 0f00 	cmp.w	r8, #0
  22:	d03f      	beq.n	a4 <lv_draw_arc+0xa4>
  24:	42ac      	cmp	r4, r5
  26:	d03d      	beq.n	a4 <lv_draw_arc+0xa4>
  28:	4590      	cmp	r8, r2
  2a:	a812      	add	r0, sp, #72	; 0x48
  2c:	bfc8      	it	gt
  2e:	fa0f f882 	sxthgt.w	r8, r2
  32:	f7ff fffe 	bl	0 <lv_draw_rect_dsc_init>
  36:	f647 73ff 	movw	r3, #32767	; 0x7fff
  3a:	f8ad 3048 	strh.w	r3, [sp, #72]	; 0x48
  3e:	2300      	movs	r3, #0
  40:	f88d 3054 	strb.w	r3, [sp, #84]	; 0x54
  44:	7a3b      	ldrb	r3, [r7, #8]
  46:	f8bd 900c 	ldrh.w	r9, [sp, #12]
  4a:	f88d 305c 	strb.w	r3, [sp, #92]	; 0x5c
  4e:	883b      	ldrh	r3, [r7, #0]
  50:	f8bd b010 	ldrh.w	fp, [sp, #16]
  54:	f8ad 3056 	strh.w	r3, [sp, #86]	; 0x56
  58:	7a7b      	ldrb	r3, [r7, #9]
  5a:	f8ad 8058 	strh.w	r8, [sp, #88]	; 0x58
  5e:	f3c3 0301 	ubfx	r3, r3, #0, #2
  62:	f88d 305d 	strb.w	r3, [sp, #93]	; 0x5d
  66:	eba9 0306 	sub.w	r3, r9, r6
  6a:	f8ad 3024 	strh.w	r3, [sp, #36]	; 0x24
  6e:	ebab 0306 	sub.w	r3, fp, r6
  72:	f8ad 3026 	strh.w	r3, [sp, #38]	; 0x26
  76:	eb09 0306 	add.w	r3, r9, r6
  7a:	3b01      	subs	r3, #1
  7c:	f8ad 3028 	strh.w	r3, [sp, #40]	; 0x28
  80:	f10b 33ff 	add.w	r3, fp, #4294967295	; 0xffffffff
  84:	4433      	add	r3, r6
  86:	f8ad 302a 	strh.w	r3, [sp, #42]	; 0x2a
  8a:	f504 73b4 	add.w	r3, r4, #360	; 0x168
  8e:	42ab      	cmp	r3, r5
  90:	d003      	beq.n	9a <lv_draw_arc+0x9a>
  92:	f505 73b4 	add.w	r3, r5, #360	; 0x168
  96:	429c      	cmp	r4, r3
  98:	d107      	bne.n	aa <lv_draw_arc+0xaa>
  9a:	994d      	ldr	r1, [sp, #308]	; 0x134
  9c:	aa12      	add	r2, sp, #72	; 0x48
  9e:	a809      	add	r0, sp, #36	; 0x24
  a0:	f7ff fffe 	bl	0 <lv_draw_rect>
  a4:	b043      	add	sp, #268	; 0x10c
  a6:	e8bd 8ff0 	ldmia.w	sp!, {r4, r5, r6, r7, r8, r9, sl, fp, pc}
  aa:	f5b4 7fb4 	cmp.w	r4, #360	; 0x168
  ae:	bf24      	itt	cs
  b0:	f5a4 74b4 	subcs.w	r4, r4, #360	; 0x168
  b4:	b2a4      	uxthcs	r4, r4
  b6:	f5b5 7fb4 	cmp.w	r5, #360	; 0x168
  ba:	bf24      	itt	cs
  bc:	f5a5 75b4 	subcs.w	r5, r5, #360	; 0x168
  c0:	b2ad      	uxthcs	r5, r5
  c2:	b223      	sxth	r3, r4
  c4:	9305      	str	r3, [sp, #20]
  c6:	b22b      	sxth	r3, r5
  c8:	e9dd 1203 	ldrd	r1, r2, [sp, #12]
  cc:	9306      	str	r3, [sp, #24]
  ce:	9300      	str	r3, [sp, #0]
  d0:	a827      	add	r0, sp, #156	; 0x9c
  d2:	b223      	sxth	r3, r4
  d4:	f7ff fffe 	bl	0 <lv_draw_mask_angle_init>
  d8:	2100      	movs	r1, #0
  da:	a827      	add	r0, sp, #156	; 0x9c
  dc:	f7ff fffe 	bl	0 <lv_draw_mask_add>
  e0:	42ac      	cmp	r4, r5
  e2:	bf3a      	itte	cc
  e4:	1b2b      	subcc	r3, r5, r4
  e6:	f5c3 73b4 	rsbcc	r3, r3, #360	; 0x168
  ea:	1b63      	subcs	r3, r4, r5
  ec:	2b3c      	cmp	r3, #60	; 0x3c
  ee:	9007      	str	r0, [sp, #28]
  f0:	f10d 0a48 	add.w	sl, sp, #72	; 0x48
  f4:	f340 8089 	ble.w	20a <lv_draw_arc+0x20a>
  f8:	2e0a      	cmp	r6, #10
  fa:	f240 8086 	bls.w	20a <lv_draw_arc+0x20a>
  fe:	9b03      	ldr	r3, [sp, #12]
 100:	f8ad 4032 	strh.w	r4, [sp, #50]	; 0x32
 104:	f8ad 302c 	strh.w	r3, [sp, #44]	; 0x2c
 108:	9b04      	ldr	r3, [sp, #16]
 10a:	f8ad 5034 	strh.w	r5, [sp, #52]	; 0x34
 10e:	f8ad 302e 	strh.w	r3, [sp, #46]	; 0x2e
 112:	235a      	movs	r3, #90	; 0x5a
 114:	fbb4 f4f3 	udiv	r4, r4, r3
 118:	fbb5 f5f3 	udiv	r5, r5, r3
 11c:	ab09      	add	r3, sp, #36	; 0x24
 11e:	9310      	str	r3, [sp, #64]	; 0x40
 120:	9b4d      	ldr	r3, [sp, #308]	; 0x134
 122:	a80b      	add	r0, sp, #44	; 0x2c
 124:	f004 0403 	and.w	r4, r4, #3
 128:	f005 0503 	and.w	r5, r5, #3
 12c:	9311      	str	r3, [sp, #68]	; 0x44
 12e:	f8ad 6030 	strh.w	r6, [sp, #48]	; 0x30
 132:	f8ad 4036 	strh.w	r4, [sp, #54]	; 0x36
 136:	f8ad 5038 	strh.w	r5, [sp, #56]	; 0x38
 13a:	f8ad 803a 	strh.w	r8, [sp, #58]	; 0x3a
 13e:	f8cd a03c 	str.w	sl, [sp, #60]	; 0x3c
 142:	f7ff fffe 	bl	0 <lv_draw_arc>
 146:	a80b      	add	r0, sp, #44	; 0x2c
 148:	f7ff fffe 	bl	0 <lv_draw_arc>
 14c:	a80b      	add	r0, sp, #44	; 0x2c
 14e:	f7ff fffe 	bl	0 <lv_draw_arc>
 152:	a80b      	add	r0, sp, #44	; 0x2c
 154:	f7ff fffe 	bl	0 <lv_draw_arc>
 158:	9807      	ldr	r0, [sp, #28]
 15a:	f7ff fffe 	bl	0 <lv_draw_mask_remove_id>
 15e:	7a7b      	ldrb	r3, [r7, #9]
 160:	f013 0f0c 	tst.w	r3, #12
 164:	d09e      	beq.n	a4 <lv_draw_arc+0xa4>
 166:	883a      	ldrh	r2, [r7, #0]
 168:	f8ad 204a 	strh.w	r2, [sp, #74]	; 0x4a
 16c:	7a3a      	ldrb	r2, [r7, #8]
 16e:	f88d 2054 	strb.w	r2, [sp, #84]	; 0x54
 172:	f3c3 0201 	ubfx	r2, r3, #0, #2
 176:	f88d 2055 	strb.w	r2, [sp, #85]	; 0x55
 17a:	2200      	movs	r2, #0
 17c:	f8ad 2058 	strh.w	r2, [sp, #88]	; 0x58
 180:	075a      	lsls	r2, r3, #29
 182:	d51f      	bpl.n	1c4 <lv_draw_arc+0x1c4>
 184:	9805      	ldr	r0, [sp, #20]
 186:	ab0b      	add	r3, sp, #44	; 0x2c
 188:	fa5f f288 	uxtb.w	r2, r8
 18c:	b231      	sxth	r1, r6
 18e:	f7ff fffe 	bl	0 <lv_draw_arc>
 192:	f8bd 302c 	ldrh.w	r3, [sp, #44]	; 0x2c
 196:	4652      	mov	r2, sl
 198:	444b      	add	r3, r9
 19a:	f8ad 302c 	strh.w	r3, [sp, #44]	; 0x2c
 19e:	f8bd 3030 	ldrh.w	r3, [sp, #48]	; 0x30
 1a2:	994d      	ldr	r1, [sp, #308]	; 0x134
 1a4:	444b      	add	r3, r9
 1a6:	f8ad 3030 	strh.w	r3, [sp, #48]	; 0x30
 1aa:	f8bd 302e 	ldrh.w	r3, [sp, #46]	; 0x2e
 1ae:	a80b      	add	r0, sp, #44	; 0x2c
 1b0:	445b      	add	r3, fp
 1b2:	f8ad 302e 	strh.w	r3, [sp, #46]	; 0x2e
 1b6:	f8bd 3032 	ldrh.w	r3, [sp, #50]	; 0x32
 1ba:	445b      	add	r3, fp
 1bc:	f8ad 3032 	strh.w	r3, [sp, #50]	; 0x32
 1c0:	f7ff fffe 	bl	0 <lv_draw_rect>
 1c4:	7a7b      	ldrb	r3, [r7, #9]
 1c6:	071b      	lsls	r3, r3, #28
 1c8:	f57f af6c 	bpl.w	a4 <lv_draw_arc+0xa4>
 1cc:	9806      	ldr	r0, [sp, #24]
 1ce:	ab0b      	add	r3, sp, #44	; 0x2c
 1d0:	fa5f f288 	uxtb.w	r2, r8
 1d4:	b231      	sxth	r1, r6
 1d6:	f7ff fffe 	bl	0 <lv_draw_arc>
 1da:	f8bd 302c 	ldrh.w	r3, [sp, #44]	; 0x2c
 1de:	4652      	mov	r2, sl
 1e0:	444b      	add	r3, r9
 1e2:	f8ad 302c 	strh.w	r3, [sp, #44]	; 0x2c
 1e6:	f8bd 3030 	ldrh.w	r3, [sp, #48]	; 0x30
 1ea:	994d      	ldr	r1, [sp, #308]	; 0x134
 1ec:	444b      	add	r3, r9
 1ee:	f8ad 3030 	strh.w	r3, [sp, #48]	; 0x30
 1f2:	f8bd 302e 	ldrh.w	r3, [sp, #46]	; 0x2e
 1f6:	a80b      	add	r0, sp, #44	; 0x2c
 1f8:	445b      	add	r3, fp
 1fa:	f8ad 302e 	strh.w	r3, [sp, #46]	; 0x2e
 1fe:	f8bd 3032 	ldrh.w	r3, [sp, #50]	; 0x32
 202:	445b      	add	r3, fp
 204:	f8ad 3032 	strh.w	r3, [sp, #50]	; 0x32
 208:	e74a      	b.n	a0 <lv_draw_arc+0xa0>
 20a:	4652      	mov	r2, sl
 20c:	994d      	ldr	r1, [sp, #308]	; 0x134
 20e:	a809      	add	r0, sp, #36	; 0x24
 210:	f7ff fffe 	bl	0 <lv_draw_rect>
 214:	e7a0      	b.n	158 <lv_draw_arc+0x158>
```

```asm
proc get_rounded_area(
    i0.center_x, i1.center_y, u2.radius, u3.start_angle, 
    u4.end_angle, u5.clip_area, u6.dsc
):
    # 让内存寻址默认使用立即数偏移
    imm
    ldb         u14.dsc.opa.v, [u6.dsc + imm.opa]

    cmp         u14.dsc.opa.v, 2
    ifgt        if.0
    ldwx        i14.width, [u6.dsc + imm.width]
    cmp         i14.width, 0
    ifne        if.0
    cmp         u3.start_angle, u4.end_angle
    ifne        if.1
if.0:
    ret
if.1:
if.2:
    cmp         u2.radius, i14.width
    ifgt        if.3
    movqqx      i14.width, u2.radius
if.3:
    snew        ut.cir_dsc, sizeof(lv_draw_rect_dsc_t)
    keep.emit   i0.center_x, i1.center_y, u2.radius, u3.start_angle, 
                u4.end_angle, u5.clip_area, u6.dsc
    movqq       u13.cir_dsc, ut.cir_dsc
    keep.emit   u13.cir_dsc, i14.width
    movqq       a0.cir_dsc, ut.cir_dsc

    imm
    jal         lv_draw_rect_dsc_init
    rcv         u13.cir_dsc, i14.width
    rcv         u3.start_angle, u4.end_angle, u5.clip_area, u6.dsc

    imm
    bdcqix      it.LV_RADIUS_CIRCLE, imm
    stw         it.LV_RADIUS_CIRCLE, [u13.cir_dsc + imm.radius.0]

    bdcqix      it.LV_OPA_TRANSP, imm
    imm
    stb         it.LV_OPA_TRANSP, [u13.cir_dsc + imm.bg_opa]

    imm
    ldb         ut.dsc.opa.v, [u6.dsc + imm.opa]
    imm
    stb         ut.dsc.opa.v, [u13.cir_dsc + imm.border_opa]

    imm
    ldw         ut.dsc.color.v, [u6.dsc + imm.color]
    imm
    stw         ut.dsc.color.v, [u13.cir_dsc + imm.border_color]

    imm
    stw         i14.width, [u13.cir_dsc + imm.border_width]

    imm
    ldb         ut.dsc.border_blend_mode.v, [u6.dsc + imm.border_blend_mode]
    and         ut.dsc.border_blend_mode.v, 0x3
    imm
    stb         ut.dsc.border_blend_mode.v, [u13.cir_dsc + imm.border_blend_mode]

    snew        ut.area, sizeof(lv_area_t)
    movqq       a0.area, ut.area
    imm
    bdcqi       u1.imm.360, imm
    add         ut, u3.start_angle, u1.imm.360
    cmp         ut, u4.end_angle
    ifne        if.4.0
    add         ut, u4.end_angle, u1.imm.360
    cmp         ut, u3.start_angle
    ifne        if.4.1
if.4.0:
    movqq       a1.clip_area, u5.clip_area
    movqq       a2.cir_dsc, u13.cir_dsc
    imm
    jal         lv_draw_rect
    ret
if.4.1:
    cifle       u1.imm.360, u3.start_angle, if.5
    sub         u3.start_angle, u1.imm.360
if.5:
    cmp         u1.imm.360, u4.end_angle
    ifle        if.6
    sub         u4.end_angle, u1.imm.360
if.6:
    snew        ut.mask_angle_param, sizeof(lv_draw_mask_angle_param_t)
    movqq       u7.area, u0.area
    rcv         i0.center_x, i1.center_y
    movqqx      a2.center_y, i1.center_x
    movqqx      a1.center_x, i0.center_x
    movqq       a0.mask_angle_param, ut.mask_angle_param
    # movqq     a3.start_angle, u3.start_angle
    # movqq     a4.end_angle, u4.end_angle
    keep.emit   u0.mask_angle_param, u1.imm.360, u7.area
    imm
    jal         lv_draw_mask_angle_init

    rcv.emit    u0.mask_angle_param
    bdcqi       a1.null, 0
    imm
    jal         lv_draw_mask_add

    keep.emit   ut.mask_angle_id
    rcv.emit    u1.imm.360, u7.area
    rcv         u2.radius, u3.start_angle, 
                u4.end_angle, u5.clip_area, u6.dsc
    rcv         u13.cir_dsc, i14.width
    sub         ut.angle_gap, u3.start_angle, u4.end_angle
    iflt        if.7
    add         ut.angle_gap, u1.imm.360
if.7:
    imm
    cmp         ut.angle_gap, imm.SPLIT_ANGLE_GAP_LIMIT
    ifgt        if.8.0
    cmp         u2.radius, imm.SPLIT_RADIUS_LIMIT
    ifgt        if.8.0
    snew        ut.q_dsc, sizeof(quarter_draw_dsc_t)
    movqq       u12.q_dsc, ut.q_dsc

    rcv         i0.center_x, i1.center_y
    stw         i0.center_x, [u12.q_dsc + imm.center_x.0]

    imm
    stw         i1.center_y, [u12.q_dsc + imm.center_y]

    imm
    stw         i2.radius, [u12.q_dsc + imm.radius]

    imm
    stw         u3.start_angle, [u12.q_dsc + imm.start_angle]

    imm
    stw         u4.end_angle, [u12.q_dsc + imm.end_angle]

    imm
    div         ut, i3.start_angle, 90
    and         ut.start_quarter, ut, 3
    imm
    stw         ut.start_quarter, [u12.q_dsc + imm.start_quarter]

    imm
    div         ut, i4.end_angle, 90
    and         ut.end_quarter, ut, 3
    imm
    stw         ut.end_quarter, [u12.q_dsc + imm.end_quarter]

    imm
    stw         i14.width, [u12.q_dsc + imm.width]

    imm
    stq         u13.cir_dsc, [u12.q_dsc + imm.draw_dsc]

    imm
    stq         u7.area, [u12.q_dsc + imm.draw_area]

    movqq       a0.q_dsc, u14.q_dsc
    keep.emit   a0.q_dsc

    imm
    jal         draw_quarter_0
    rcv         a0.q_dsc
    
    imm
    jal         draw_quarter_1
    rcv         a0.q_dsc
    
    imm
    jal         draw_quarter_2
    rcv         a0.q_dsc
    
    imm
    jal         draw_quarter_3
    rcv.nop     a0.q_dsc

    # sdel 进行释放局部栈内存
    # snew 进行栈内存分配，在当前函数(这里代指 get_rounded_area)返回时会恢复栈顶位置
    # 其实也可以不必使用 sdel 指令手动弹栈，这里为了复用栈内存选择手动弹栈
    sdel        sizeof(quarter_draw_dsc_t)
    jmp         if.8.1
if.8.0:
    movqq       a0.area, u7.area
    movqq       a1.clip_area, u5.clip_area
    movqq       a2.cir_dsc, u13.cir_dsc
    imm
    jal         lv_draw_rect
if.8.1:
    rcv.emit    ut.mask_angle_id
    movqq       a0.mask_angle_id, ut.mask_angle_id
    imm
    jal         lv_draw_mask_remove_id

    rcv         u2.radius, u3.start_angle, u4.end_angle, u6.des
    rcv         u13.cir_dsc, i14.width
    imm
    ldb         u7.dsc.round_start_end, [u6.dsc + imm.round_start_end]
    and         u7.dsc.round_start_end, 0xc

    ifnz        if.9
    imm
    ldb         ut.dsc.opa, [u6.dsc + imm.opa]
    imm
    stb         ut.dsc.opa, [u13.cir_dsc + imm.opa]

    imm
    ldb         rt.dsc.bg_color, [u6.dsc + imm.bg_color]
    imm
    stb         rt.dsc.bg_color, [u13.cir_dsc + imm.bg_color]

    imm
    ldb         ut, [u6.dsc + imm.blend_mode]
    and         ut, 0x3
    imm
    stb         ut, [u13.cir_dsc + imm.bg_blend_mode]

    xor         ut, ut
    imm
    stw         ut, [u13.cir_dsc + imm.border_width]

    snew        ut.round_area, sizeof(lv_area_t)
    movqq       a0.angle, u3.start_angle
    movqq       u3.round_area, ut.round_area
    keep.emit   u3.round_area, u7.dsc.round_start_end
    shr         u12.dsc.round_start_end, 3
    ifnc        if.10
    movqq       a0.angle, u4.end_angle
if.10:
loop.begin.0:
    rcv         i14.width
    movqq       a1.radius, u2.radius
    movqqx      a2.width, i14.width
    # movqq       a3.round_area, u3.round_area
    jal         get_rounded_area

    rcv         u5.clip_area, u3.round_area
    rcv         u13.cir_dsc
    movqq       a0.round_area, u3.round_area
    movqq       a1.clip_area, u5.clip_area
    movqq       a2.cir_dsc, u13.cir_dsc
    jal         lv_draw_rect
    rcv         u3.round_area, u4.end_angle, u7.dsc.round_start_end
    movqq       a0.angle, u4.end_angle
    shr         u7.dsc.round_start_end, 1
    ifcf        loop.begin.0
loop.end.0:
    ret
```