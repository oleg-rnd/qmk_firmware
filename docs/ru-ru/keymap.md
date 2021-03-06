# Принципы работы прошивки

> **Warning:**
> Перевод не профессиональный, и улучшения приветствуются!

Раскладки в QMK описываются исходнм файлом с расширением ".C". Структура данных в нём - это массив массивов. Внешний массив - это список массивов слоев, а внутренний массив - "список" клавиш(keys).  
Большинство клавиатур используют макрос LAYOUT(), чтобы создать этот массив массивов.  

## Раскладка и слои 

В QMK константа `const uint16_t PROGMEM keymaps[][MATRIX_ROWS][MATRIX_COLS]` содержит информацию о слоях раскладок(keymaps) клавиатуры в 16-битном формате, и "команды", или "коды", для исполнения(action codes). 
Ограничение: слоев может быть не более 32-х. 

Для отражения нажатой клавиши на уровне сигнала, обычно, все старшие 8 бит в action code(коде команды) есть 0, а "младшие" 8 бит содержат код использования USB HID, а точнее кей-код сгенерированный для клавиши.  
Корреспондирующие (т.е. одновременно активные) слои клавиатуры могут считываться одновременно. Они индексируются номерами от 0 до 31, и более высокий уровень имеет приоритет.


    Keymap: 32 Layers                   Layer: action code matrix
    -----------------                   ---------------------
    stack of layers                     array_of_action_code[row][column]
           ____________ precedence               _______________________
          /           / | high                  / ESC / F1  / F2  / F3   ....
      31 /___________// |                      /-----/-----/-----/-----
      30 /___________// |                     / TAB /  Q  /  W  /  E   ....
      29 /___________/  |                    /-----/-----/-----/-----
       :   _:_:_:_:_:__ |               :   /LCtrl/  A  /  S  /  D   ....
       :  / : : : : : / |               :  /  :     :     :     :
       2 /___________// |               2 `--------------------------
       1 /___________// |               1 `--------------------------
       0 /___________/  V low           0 `--------------------------


Иногда код каманды(action code), находящийся в раскладке(keymap), в некоторых файлах, может называться кей-кодом, это наследие прошивки TMK. 

### Определение текущего слоя

Определение текущего слоя раскладки зависит от двух 32-битных параметров:  

* **default_layer_state** - указывает базовый слой раскладки клавиатуры (0-31), который всегда(?) действителен и на который следует ссылаться (уровень по умолчанию). 
* **layer_state**         - своим битом отражает текущий статус включения/выключения каждого слоя (если бит равен "1", то слой активен).  

Слой раскладки, обозначенный как «0», обычно является default_layer, остальные слои изначально отключены после загрузки прошивки, хотя это можно настроить по-другому в config.h.  
Значение default_layer полезно изменять при полном переключении раскладки клавиш, например, если вы хотите переключиться на Colemak вместо Qwerty.  


    Initial state of Keymap          Change base layout
    -----------------------          ------------------

      31                               31
      30                               30
      29                               29
       :                                :
       :                                :   ____________
       2   ____________                 2  /           /
       1  /           /              ,->1 /___________/
    ,->0 /___________/               |  0
    |                                |
    `--- default_layer = 0           `--- default_layer = 1
         layer_state   = 0x00000001       layer_state   = 0x00000002

С другой стороны, вы можете изменить layer_state, чтобы наложить базовый слой на другие слои для таких функций, как клавиши навигации, функциональные клавиши (F1-F12), мультимедийные клавиши и/или другие действия.

    Overlay feature layer
    ---------------------      bit|status
           ____________        ---+------
      31  /           /        31 |   0
      30 /___________// -----> 30 |   1
      29 /___________/  -----> 29 |   1
       :                        : |   :
       :   ____________         : |   :
       2  /           /         2 |   0
    ,->1 /___________/  ----->  1 |   1
    |  0                        0 |   0
    |                                 +
    `--- default_layer = 1            |
         layer_state   = 0x60000002 <-'


### Приоритетность и "прозрачность" слоев

Обратите внимание, что более высокие уровни имеют более высокий приоритет в "стеке" слоев.  
Прошивка "спускается вниз" с верхних активных слоев к нижним для "поиска" активного кей-кода.  
Как только она "видит" кейкод, отличный от KC_TRNS ("прозрачный") в активном слое, она прекращает поиск, и указания на более низкие уровни не выполняются.  

           ____________
          /           /  <--- Higher layer
         /  KC_TRNS  //
        /___________//   <--- Lower layer (KC_A)
        /___________/
    
На примере ниже "непрозрачные" кейкоды на более высоком слое будут исполняться, но всякий раз, когда на активном слое считывается «KC_TRNS» (или эквивалент), будет использоваться кей-код («KC_A»), находящийся ниже.  

Варианты обозначения "прозрачности":
* `KC_TRANSPARENT`
* `KC_TRNS` (alias)
* `_______` (alias)  
Эти кейкоды позволяют прошивке переходить на нижние уровни в поисках "непрозрачного" кейкода для использования.

## "Анатомия" keymap.c

В этом примере рассмотривается [старая, "дефолтная", версия раскладки(keymap)](https://github.com/qmk/qmk_firmware/blob/ca01d94005f67ec4fa9528353481faa622d949ae/keyboards/clueboard/keymaps/default/keymap.c) для клавиатуры Clueboard 66%. 

В файле keymap.c есть 3 основных раздела, о которых вам нужно знать:

- [The Definitions[En]](https://docs.qmk.fm/#/keymap?id=definitions) - "Определения"
- [The Layer/Keymap Datastructure[En]](https://docs.qmk.fm/#/keymap?id=layers-and-keymaps)  - Структура данных слоёв/раскладок клавиатуры
- [Custom Functions[En]](https://docs.qmk.fm/#/keymap?id=custom-functions), if any - Пользовательские функции, если они есть
  
### Definitions ("Определения")

Вверху файла вы найдете:

    #include QMK_KEYBOARD_H

    // Helpful defines
    #define GRAVE_MODS  (MOD_BIT(KC_LSHIFT)|MOD_BIT(KC_RSHIFT)|MOD_BIT(KC_LGUI)|MOD_BIT(KC_RGUI)|MOD_BIT(KC_LALT)|MOD_BIT(KC_RALT))

    /* * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * 
     *  You can use _______ in place for KC_TRNS (transparent)   *
     *  Or you can use XXXXXXX for KC_NO (NOOP)                  *
     * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * */

    // Each layer gets a name for readability.
    // The underscores don't mean anything - you can
    // have a layer called STUFF or any other name.
    // Layer names don't all need to be of the same
    // length, and you can also skip them entirely
    // and just use numbers.
    #define _BL 0
    #define _FL 1
    #define _CL 2

Это - определения, которые мы можем использовать при создании раскладки клавиатуры и нашей пользовательской функции.  
Определение GRAVE_MODS будет использоваться позже в нашей пользовательской функции, а следующие определения _BL, _FL и _CL (Base Layer (Default Layer), Function Layer, Control layer) упрощают обращение к каждому из наших слоев.

Примечание: Вы также можете обнаружить, что некоторые старые файлы раскладки клавиатуры также могут иметь в качестве определения значения "_______" и/или "XXXXXXX".  
Их можно использовать вместо "KC_TRNS" и "KC_NO" соответственно, чтобы было лучше видно, какие значения клавиш "перекрывает слой". Эти "определения" теперь не нужны, т.к. они задействованы по умолчанию.

### Слои и раскладки в них

Основная часть этого файла - определение keymaps []. Здесь вы перечисляете свои слои и их содержимое. Эта часть файла начинается с 
`const uint16_t keymaps PROGMEM[][MATRIX_ROWS][MATRIX_COLS] = {`

После этого вы найдете список макросов LAYOUT ().  
LAYOUT () - это просто список значений клавиш для одного слоя. Обычно у вас есть один или несколько «базовых слоев» (таких как QWERTY, Dvorak или Colemak), а затем вы накладываете слой поверх этого одного или нескольких «функциональных» слоев. Из-за способа обработки слоев вы не можете накладывать «нижний» слой поверх «более высокого» слоя.

`keymaps[][MATRIX_ROWS][MATRIX_COLS]` в QMK содержит в себе 16-битный action code. 
В соответствии, с тем, что сказано выше, в кейкоде, представляющем "типовые" клавиши, старший байт равен 0, а младший байт - это идентификатор USB HID usage ID для клавиатуры. 

> TMK, из которого был форкнут QMK, использует вместо этого `const uint8_t PROGMEM keymaps[][MATRIX_ROWS][MATRIX_COLS]` и содержит 8-битный кейкод.
> Некоторые значения кейкодов зарезервированы, чтобы вызывать выполнение определенных кодов действий(action codes) через массив fn_actions [].

#### Базовый слой

Пример базового слоя Clueboard:

      /* Keymap _BL: Base Layer (Default Layer)
       */
    [_BL] = LAYOUT(
      F(0),    KC_1,    KC_2,   KC_3,   KC_4,   KC_5,   KC_6,   KC_7,   KC_8,   KC_9,    KC_0,     KC_MINS,  KC_EQL,   KC_GRV,  KC_BSPC,          KC_PGUP, \
      KC_TAB,  KC_Q,    KC_W,   KC_E,   KC_R,   KC_T,   KC_Y,   KC_U,   KC_I,   KC_O,    KC_P,     KC_LBRC,  KC_RBRC,  KC_BSLS,                   KC_PGDN, \
      KC_CAPS, KC_A,    KC_S,   KC_D,   KC_F,   KC_G,   KC_H,   KC_J,   KC_K,   KC_L,    KC_SCLN,  KC_QUOT,  KC_NUHS,  KC_ENT,                             \
      KC_LSFT, KC_NUBS, KC_Z,   KC_X,   KC_C,   KC_V,   KC_B,   KC_N,   KC_M,   KC_COMM, KC_DOT,   KC_SLSH,  KC_RO,    KC_RSFT,          KC_UP,            \
      KC_LCTL, KC_LGUI, KC_LALT, KC_MHEN,          KC_SPC,KC_SPC,                        KC_HENK,  KC_RALT,  KC_RCTL,  MO(_FL), KC_LEFT, KC_DOWN, KC_RGHT),

Несколько интересных моментов:  
С точки зрения исходного кода C это всего лишь один массив, но в коде встроены пробелы, чтобы было лучше видно, где находится каждая клавиша на физическом устройстве.  
Обычные сканкоды клавиатуры имеют префикс KC_, а «специальные» клавиши, например, модификаторы - нет.  
Верхняя левая клавиша в данной раскладке активирует пользовательскую функцию 0 (F (0))  
Клавиша «Fn» определяется с помощью MO (_FL), который перемещается на уровень _FL, пока эта клавиша удерживается нажатой.
     
#### Функциональный (расположенный "поверх") слой

Функциональный слой "с точки зрения кода" не отличается от базового слоя.  
Но, теоретически, этот слой наложение, а не замена.  
Это может покажется незначительным различием, но по мере того, как вы создаете более сложные настройки слоев, оно будет все более и более важным.     

    [_FL] = LAYOUT(
      KC_GRV,  KC_F1,   KC_F2,  KC_F3,  KC_F4,  KC_F5,  KC_F6,  KC_F7,  KC_F8,  KC_F9,   KC_F10,   KC_F11,   KC_F12,   _______, KC_DEL,           BL_STEP, \
      _______, _______, _______,_______,_______,_______,_______,_______,KC_PSCR,KC_SLCK, KC_PAUS,  _______,  _______,  _______,                   _______, \
      _______, _______, MO(_CL),_______,_______,_______,_______,_______,_______,_______, _______,  _______,  _______,  _______,                           \
      _______, _______, _______,_______,_______,_______,_______,_______,_______,_______, _______,  _______,  _______,  _______,          KC_PGUP,         \
      _______, _______, _______, _______,        _______,_______,                        _______,  _______,  _______,  MO(_FL), KC_HOME, KC_PGDN, KC_END),

Примечание:  
- Здесь использовано обозначение `"_______"`, чтобы превратить KC_TRNS в `"_______"`.  
- Это упрощает поиск значений, которые изменились в этом слое (в сравнении с нижеидущим).  
- Находясь в этом слое, если вы нажмете одну из клавиш _______, и она активирует клавишу на следующем нижнем активном слое.

Смотрите ещё:  
- [Кей-коды[En]](https://docs.qmk.fm/#/keycodes)  
- [FAQ](https://docs.qmk.fm/#/faq_keymap) по раскладке клавиатуры
     
[Исходный документ](https://github.com/qmk/qmk_firmware/blob/master/docs/keymap.md)
