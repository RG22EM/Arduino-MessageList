# Arduino-MessageList

Другая идея, прямо вытекает из https://github.com/DetSimen/Arduino-. Дело в том, что процедура срабатывания таймера выполняется в контексте прерывания, поэтому не может быть длинной, да и вапще, нехорошо это.  Поэтому, придумывать я, канешно, ничего не стал, просто взял идею из всеми нами любимой Виндовс. Нужно создать на Ардуине простенькую, плохонькую, но очередь сообщений.  Каждый датчик меряет какие-то значения и генерирует какие-то события. Если ждать их наступления в loop(), то эта нещасная функция в оконцовке вырастает до неприемлемо огромных значений, а, как мы помним из детсадовского курса погроммирования, если текст фунции не помещается в один экран - этта очень плохой стиль программирования, надо разбивать ее на более мелкие части.  Поэтому, для своих проектов я применяю достаточно простое решение, существенно облегчающее (мне) жизнь. 

Для начала определим, какие сообщения мы хотим видеть в своей программе, записываем их в файл DEF_Message.h. Файл имеет саму обчную простецкую структуру, ничего, кроме #define тама и нет. 

  #pragma once

  // system messages

  #define  msg_Empty 0x00
  #define  msg_Error 0x01
  #define  msg_Paint 0x02

  // user messages

  #define  msg_SecondTick		0x10  // секунда тикнула
  #define	 msg_ReadMQ2			0x12	//читать данные с датчика MQ2
  #define  msg_MQ2Changed		0x13	// показания c MQ2 изменились
  #define  msg_ReadKey			0x14  // читать клавиатуру

  #define  msg_KeyDown			0x20	// кнопка нажата
  #define	 msg_KeyUp				0x21	// кнопка отпущена

  #define msg_VentON				0x22	// включить вытяжку
  #define msg_VentOFF				0x23	// выключить вытяжку

Просто определяем числовые значения для всех используемых нами сообщений 0-255 (1 байт). Смотрим только, чтобы разные сообщения имели разные номера, иначе аяяй (будет всегда выполняться функция первого сообщения)


первые 16 сообщений я выделил как системные, разницу между ними и другими расскажу позже.
