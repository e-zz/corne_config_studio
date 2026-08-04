# AGENTS.md — corne_config_studio

ZMK 固件配置仓库。同时构建**两个键盘**，共用同一个 keymap 文件：
- Corne（nice_nano_v2，左右分体）
- **Chipper42**（seeeduino_xiao_ble，自定义 shield，42 键 6 列分体）

GitHub Actions 自动编译。本文件记录仓库结构、keymap 布局约定和注意事项，避免重复全量扫描。

## 构建

- CI：`.github/workflows/build.yml` → 复用 `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`，push/PR 触发。
- 产物矩阵见 `build.yaml`（两套键盘都有）：
  - Corne：`nice_nano_v2` + `corne_left`(带 studio) / `corne_right` / `settings_reset`
  - Chipper42：`seeeduino_xiao_ble` + `chipper42_left`(带 studio) / `chipper42_right` / `settings_reset`
- ZMK 版本固定：`config/west.yml` → `revision: v0.3`。**不要动版本锁定**。
- 本地无 west/Zephyr 工具链，**验证方式 = push 后看 GitHub Actions 是否编译通过**。

## 文件地图

| 路径 | 作用 |
|------|------|
| `config/corne.keymap` | **唯一核心文件**。所有按键布局。两键盘共用。 |
| `config/chipper42.keymap` | 仅 `#include "corne.keymap"`，无实际内容 |
| `config/corne.conf` | Corne 板级配置（键盘名 "Corner"、OLED、BT 功率、指针、去抖） |
| `config/chipper42.conf` | Chipper42 板级配置（键盘名 "chipper"，结构同 corne.conf） |
| `config/boards/shields/chipper42/` | **Chipper42 自定义 shield 定义**（dtsi/overlay/Kconfig，勿乱动） |
| `config/west.yml` | ZMK 版本锁定 v0.3 |
| `build.yaml` | CI 构建矩阵（两套键盘） |
| `zephyr/module.yml` | ZMK module 声明 |

### Chipper42 shield（重点文件）

- `chipper42.dtsi` — kscan 矩阵、`default_transform`（42 键，同 Corne 编号）、OLED 显示、physical layout
- `chipper42_left.overlay` / `chipper42_right.overlay` — 左右半 col-gpios；右半用 `col-offset = <6>` 映射右半键位
- `chipper42_left.conf` — NFC pin 复用为 GPIO + 显示
- `Kconfig.defconfig` / `Kconfig.shield` / `chipper42.zmk.yml` — shield 声明（左右半、BLE 角色、LVGL/SSD1306 选项）

## Keymap 结构（corne.keymap）

42 键。层 = keymap 节点内 layer 子节点，**层索引 = 声明顺序**，改顺序会破坏 `&tog N`/`&lt N`/`&lm N`/combo `layers` 引用。

层顺序（当前索引，与 main 分支**不同**，本分支多一个 fast 层）：
0. `default_layer` — QWERTY 主层
1. `symbol_layer` — "M" 符号层
2. `num_layer` — "N" 数字层
3. `fun_layer` — "Fn" 功能层（F 键/BT/音量，第一个键是 `&kp LC(LA(HOME))`）
4. `sys_layer` — "SYS"（电源/背光/bootloader/`studio_unlock`）
5. `kp_layer` — "K" 数字键盘 + 鼠标层（`&mmv`/`&msc`/`&mkp`）
6. `fast_kp_layer` — "FK" 全 `&trans`，**瞬时精度层**：由 `&lm 6 LCTRL` 触发，且 `mmv_input_listener` 的 `precision_mode` 监听 `layers = <6>`（x2 速度）
7. `shift_layer` — "R1"：QWERTY 但**右手半区所有列右移一列**（col5 回卷到 col0），左半/拇指不变。切换键 = 右拇指最右（pos 41，`&tog 7`，原 `&kp RCTRL`）
8. `symbol_shift_layer` — "MR"：symbol 的旋转版（右手半区右移一列）
9. `num_shift_layer` — "NR"：num 的旋转版
10. `fun_shift_layer` — "FR"：fun 的旋转版
11. `sys_shift_layer` — "SR"：sys 的旋转版
12. `kp_shift_layer` — "KR"：kp 的旋转版

**层 8-12（旋转变体）**：shift 模式下进入的所有层都必须是旋转版，否则右手错位一列。R1 上的层入口全部指向变体（pos 19 hold→KR、pos 39/13 hold→MR、pos 40 hold→FR、pos 32 hold→NR、combo→各 *_shift）。变体内部出口也旋转重指（`&tog NUM_S`/`&tog PAD_S`/`&tog FUN_S`），pos 41 = `&to 0`（一键回家）。fast 层(6)全 `&trans`，从 KR 里 `&lm 6 LCTRL` 直接复用，无需旋转版。

### 物理键位编号（42 键，Corne 与 Chipper42 一致）

- 左半：行0-2 = 位置 0-17，拇指 = 18,19,20
- 右半：行0-2 = 位置 21-38，拇指 = 39,40,41
- 行顺序自上而下，列顺序从左到右。
- 右半拇指：39 = `&lt SYMBOL ENTER`，40 = `&lt 3 SPACE`，**41 = `&tog 7`**（shift_layer 切换键，原为 `&kp RCTRL`）。

### 旋转规则（shift_layer 及全部 *_shift 变体，必须与 base 保持同步）

右半 6 列内容整体右移一列，最右列回卷到最左列：
- 新 col0 ← 旧 col5，新 col1 ← 旧 col0，…，新 col5 ← 旧 col4
- 左半与拇指行不变（拇指的层入口键除外：指向旋转版目标）
- 改任何 base 层右半时，必须同步更新其 *_shift 变体，否则布局脱节
- 变体内部对层号的引用（`&tog`/`&lt`）随旋转重指：出口键跟列走（如 num 的 `&tog 2` 在 pos 31 → num_shift 的 `&tog NUM_S` 在 pos 32），拇指上 `&tog 3`→`&tog FUN_S`、`&tog 5`→`&tog PAD_S`；变体 pos 41 一律 `&to 0`（一键退出 shift 世界）

### 默认层右手区列映射（基准）

| 列 | 行0 | 行1 | 行2 |
|----|-----|-----|-----|
| 0 | Y | H | B |
| 1 | U | J | N |
| 2 | I | K | M |
| 3 | O | L | `,` |
| 4 | P | `;`(`&lt 2 SEMI`) | `.` |
| 5 | BKSP(`&bsdel`) | `'` | `/`(`&mt RSHFT FSLH`) |

shift_layer 旋转后行0 = BKSP Y U I O P，行1 = `'` H J K L `;`，行2 = `/` B N M `,` `.`。

## 指针/鼠标配置（本分支内联在 keymap，无 mouse.dtsi）

- 文件顶部 `ZMK_POINTING_DEFAULT_MOVE_VAL`/`SCRL_VAL`
- `&mmv_input_listener`：`input-processors = <&zip_xy_scaler 1 1>`，并定义 `precision_mode`（`layers = <6>` → x2 速度）
- 层 5（kp_layer）右手区大量用 `&mmv`/`&msc`/`&mkp`，拇指行是鼠标三键：`&mkp LCLK`/`&mkp RCLK`/`&mkp MCLK`
- 层 6（fast）是瞬时精度模式，靠 `&lm 6 LCTRL`（见下）

## 自定义 behaviors（`/behaviors` 节点）

- `lm` — macro-two-param：`&mo` 层 + 修饰键同时按住（类似 QMK `LM()`），用法 `&lm 6 LCTRL`。**层号是参数，改层序要核查**
- `esctab` — mod-morph：ESC/TAB（带 shift 类修饰时 TAB）
- `bsdel` — mod-morph：BSPC/DEL（shift 时 DEL）
- `sftcaps` — tap-dance：SHIFT/SHIFT/CAPSLOCK（300ms）
- `lralt` — tap-dance：LALT/RALT/RA(GRAVE)（300ms）
- `u_lt` — hold-tap：hold=`&mo` / tap=`&kp`，tap-preferred，300ms（如 `&u_lt PAD SPACE`）

可跨层复用；新增层不需要新 behavior。

## Combos

| combo | 位置 | 绑定 | 层限制 |
|-------|------|------|--------|
| combo_esc | 0,1 | TAB | 仅0 |
| combo_del | 10,11 | DEL | 全部 |
| combo_tognum | 12,13 | tog 2 | **0-6** |
| combo_togsys | 0,24,12 | tog 4 | **0-6** |
| combo_end | 22,34 | END | **0-6** |
| kp_layer | 23,35 | tog 5 | **0-6** |
| combo_def_layer | 38,39 | to 0 | **0-6** |
| combo_mouse_layer | 25,13 | tog 5 | **0-6** |
| combo_tognum_shift | 12,13 | tog 9 | **7-12** |
| combo_togsys_shift | 0,25,12 | tog 11 | **7-12** |
| combo_end_shift | 23,35 | END | **7-12** |
| kp_layer_shift | 24,36 | tog 12 | **7-12** |
| combo_def_layer_shift | 33,39 | to 0 | **7-12** |
| combo_mouse_layer_shift | 26,13 | tog 12 | **7-12** |
| down/up/left/right | 31,32,33,34 | KP 数字 | 仅5 |
| num_layer_eq | 15,16 | EQUAL | 仅2 |

**限制到 0-6 的原因**：这些 combo 的物理键在 shift_layer(7) 下含义改变（如 23+35 从 I+, 变成 U+N），会误触发。**`*_shift` 变体是字母忠实版**：位置取旋转后的字母位置（如 R1 上 O 在 pos 25，所以 togsys_shift 是 0,25,12），绑定指向旋转层，只在层 7-12 生效。改 base combo 位置时同步改其 shift 版。

## ZMK Studio

- 左侧固件启用（corne_left 与 chipper42_left 都带 studio-rpc-usb-uart）。
- v0.3 中层数 = keymap 层节点数，层状态 `uint32_t`（最多 32 层），**新增层会被 studio 自动识别**，无需额外 Kconfig。
- `studio_unlock` 绑定在 sys_layer。
- 经 studio 改过层后，布局以 studio 保存的为准；本仓库 keymap 修改需重新编译刷入。

## Gotchas / 已知陷阱

1. **注释 ≠ 实际绑定**：default_layer 顶部的 ASCII 注释与真实 bindings 有多处不一致（注释的 CTRL/SHFT/ALT 与绑定错位）。**一切以 bindings 为准**，改布局时别参考注释。
2. **层索引脆弱**：`&tog 5`、`&lt 2 SEMI`、`&lm 6`、combo `layers`、`precision_mode` 的 `layers = <6>` 全按声明顺序编号。在 keymap 中间插入/删除层节点会改号，需全局核查。层 7-12 的引用分散在 shift 层、6 个 *_shift combo、变体内部出口——改层序先 grep 全部 `&tog|&lt|&lm|&u_lt|&mo|layers =`。
3. **41 位置**：物理上是"ALT"键（注释如此），但绑定原来是 RCTRL，现为 `&tog 7`。**在层 8-12（旋转变体）里 pos 41 是 `&to 0`（一键回 default）**——ALT 键在 shift 世界里永远是"退出"。
4. **层 6 已被 fast 层占用、层 7 是 shift 层**：旋转变体必须从层 8 开始，不能插在 1-7 中间。
5. **去抖**：`corne.conf`/`chipper42.conf` 设了 press 8ms / release 4ms。
6. Chipper42 是 seeeduino_xiao_ble（非 nRF52 常规板），OLED 用 I2C 引脚 4/5；改 shield 需小心 GPIO。

## 标准流程

1. 改 `config/corne.keymap`（两个键盘同时生效）
2. 保持行内 12+12 主键、6 拇指，列对齐风格与现有层一致
3. push → GitHub Actions 编译验证（两套键盘都会编）→ 成功即算通过（无法本地编译）
4. 若改 default 右半，同步 shift_layer
