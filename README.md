# Arduino-MessageList

Другая идея, прямо вытекает из https://github.com/DetSimen/Arduino-. Дело в том, что процедура срабатывания таймера выполняется в контексте прерывания, поэтому не может быть длинной, да и вапще, нехорошо это.  Поэтому, придумывать я, канешно, ничего не стал, просто взял идею из не всеми нами любимой Виндовс. Нужно создать на Ардуине простенькую, плохонькую, но очередь сообщений.  Каждый датчик меряет какие-то значения и генерирует какие-то события. Если ждать их наступления в loop(), то эта несчасная функция в оконцовке вырастает до неприемлемо огромных значений, а, как мы помним из детсадовского курса погроммирования, если текст фунции не помещается в один экран - значит ее написала очень плохая киса, неумная, и функцию надо разбивать на более мелкие части, а кису - по спине лопатой.  Поэтому, для своих проектов я применяю достаточно простое решение, существенно облегчающее (мне) жизнь. 

Для начала определим, какие сообщения мы хотим видеть в своей программе, и запишем их в файл DEF_Message.h. Файл имеет самую обычную простецкую структуру, ничего, кроме #define там нет. 

    #pragma once

	// коды используемых сообщений, размер - byte

     // system messages

    #define  msg_Empty 0x00
    #define  msg_Error 0x01
    #define  msg_Paint 0x02

    // user messages

    #define  msg_SecondTick		        0x10  // секунда тикнула
    #define  msg_ReadMQ2			0x12	//читать данные с датчика MQ2
    #define  msg_MQ2Changed		        0x13	// показания c MQ2 изменились
    #define  msg_ReadKey			0x14  // читать клавиатуру

    #define  msg_KeyDown			0x20	// кнопка нажата
    #define  msg_KeyUp				0x21	// кнопка отпущена

    #define msg_VentON				0x22	// включить вытяжку
    #define msg_VentOFF				0x23	// выключить вытяжку

Просто определяем числовые значения для всех используемых нами сообщений 0-255 (1 байт). Смотрим только, чтобы разные сообщения имели разные номера, иначе аяяй (будет всегда выполняться функция обработки первого сообщения)


первые 16 сообщений я выделил как системные, разницу между ними и другими расскажу позже.

Теперь о том, что есть сообщение.  Сообщение, сопсно, это структура описанная как 

    struct TMessage
    {
    public:
	    byte   Message;     // код сообщения, константа, описанная в DEF_Message.h 
	    word   LoParam;     // два необязательных параметра, младший 
	    word   HiParam;     // и старший 

	    TMessage();         // конструктор, создает системное пустое сообщение msg_Empty 
	    TMessage(byte msg, word loparam, word hiparam = 0);  // конструктор. Полностью создает сообщение со всеми параметрами
	    TMessage(byte msg); // конструктор. создает сообщение с кодом msg и обоими параметрами равными нулю

	    bool operator ==(TMessage msg);  // переопределенный оператор сравнения. С другим сообщением
	    bool operator ==(PMessage msg);  // с указателем на сообщение
	    bool operator ==(byte msg);      // и просто с числом (с кодом сообщения)  (для внутреннего использования)
    };
    
Далее.  Одно сообщение малоинтересно, нужно собрать их в очередь и по очереди обрабатывать.  
Следующий класс, TMessageList, является агрегатором записей типа TMessagе, грубо говоря - массивом, 
организованным в очередь, первым пришел - первым выйдешь (обработаешься).  выглядит это кактотак: 
    
    class TMessageList
    {
      private:
	    PMessage *Items;        // массив указателей на сообщения

	    byte	flength;		// емкость очереди (максимальное число сообщений в очереди) задается в конструкторе
	    byte	fcount;			// счетчик сообщений, которые находятся в очереди
	    void	DeleteFirst(void);	// удалить первое сообщение, со сдвигом остальных вперед
	    bool	FindMessage(PMessage msg);  //проверить, есть ли такое сообщение в очереди
    public:

	    TMessageList();                 // конструктор.  По умолчанию максимальная ёмкость очереди - 16 сообщений
	    TMessageList(byte length);      // а тут емкость можно задать самому (макс значение - 255, но память сломаеца гораздо раньше) 

	    bool Availiable();		// проверка, что очередь сообщений не пуста  (fcount>0)

	    TMessage GetMessage();	// взять сообщение из очереди

	    bool Add(byte msg, word lo, word high);  // добавить полностью описанное сообщение в очередь, с кодом и параметрами
	    bool Add(byte msg);		// добавить в очередь сообщение с кодом msg и обоими параметрами == 0
	    bool AddEmpty(void);	// добавить в очередь пустое сообщение (msg_Empty, 0, 0)
	    bool Paint(void);		// добавить в очередь сообщение перерисовать экран (msg_Paint,0,0)	
	    bool Error(int errornum);	// добавить в очередь системное сообщение об ошибке, errornum - номер ошибки
	    				// аналог Add(msg_Error, errornum, 0)
	    bool Error() { return Add(msg_Error); } // добавить в очередь системное сообщение об ошибке с обоими параметрами == 0

	    byte Count();		// отдает число сообщений в очереди, которые ждут обработки

    };


Например.  В сетапе создаем таймер, который будет тикать раз в секунду (часы будем строить) 

	byte hour, minutes, seconds = 0;   // глобальные переменные для времени
	THandle hClockTimer; 
	
	void setup()
	{
	 MessageList=new TMessageList();      // очередь глубиной 16 сообщений	
	 TimerList.AddSeconds(tmrClock,1);    // функция tmrClock будет вызываца 1 раз в секунду
	}

	void tmrClock(void)
	{
	 seconds++;
	 if (seconds<60) return; // так как мои часы показывают только часы и минуты, если меняются только секунды - выходим
	 seconds=0; minutes++;
	 if (minutes>59){minutes=0; hour++;};
	 if (hour>23) hour = 0;
	 MessageList.AddPaint(); 	// добавляем в очередь сообщение msg_Paint, экран надо перерисовать, 
	 				// потому что изменились показания часов или минут
	}
	
	
Вот чем отличаются системные сообщения - для них у очереди сообщений есть специальные функции для быстрого помещения такого сообщения в очередь

а в loop() начинаем крутить бесконечную карусель

	void loop() 
	{

		if (NOT MessageList->Availiable()) return; 	// если очередь пуста - выходим
		TMessage msg = MessageList->GetMessage();	// иначе - берем первое сообщение и обрататываем	

		switch (msg.Message)
		{
		case msg_Empty:					break;   // пустое сообщение, просто убираем его из очереди
		case msg_Paint:      evtPaint();		break;   // перерисовать икранчег
		default: Serial << "Error in MessageList\n";
		}
	}

	void evtPaint()
	{
	// сопсно, здесь и перерисовываем экран
	}
	
так как сообщения могут иметь параметры, можно в функцию обработки сообщения передавать эти параметры для последующей обработки

применение: 
	скачать 3 файла, 2 - h, 1 - срр.
	
	бросить в папку с проектом, подключить хидеры. 
	
	обьявить глобально extern TMessageList *MessageList;
	
	в сетапе написать MessageList = new TMessageList(); 
	
	Если очередь планируеца больше 16 сообщений, написать MessageList = new TMessageList(ваше число);
	
	дописать в DEF_Message.h коды ваших сообщений.
	
	в функции срабатывания таймеров добавлять ваши сообщения MessageList->Add(msg_xxx, loparam, hiparam) 
	
	в loop() добавлять в оператор switch обработку своих сообщений через case msg_xxx: ..... break;

Наслаждаца краткостью. 

Важно помнить, если в очереди может быть только одно сообщение с заданным кодом, другое с таким же не добавляется, пока предыдущее не обработано. Допустим, если много датчиков посылают команду перерисовки msg_Paint, то выполнять все бессмысленно, достаточно выполнить 1 раз, чтобы экранчик не моргал почем зря. Поэтому первый msg_Paint помещается в очередь и ждет обработки, а последующие просто отбрасываются.  (может быть, переделаю, чтобы дублирующее собщение помещалось в конец очереди, а предыдущее выкидывалось из очереди без обработки. Так мы будем сдвигать по времени частые или долгие операции, они обрабатывались позже).  

Пока всё, потом еще допишу.  И прошу прощенья, я хоть и связан с программированием, но язык C++ мне не родной. Сорри. 

Чуть попозже, когда найду и прокомментирую всё, выложу проект управления небольшой теплицей (форточки, полив + NRF24) и разберу по шагам доконально, что и как делается.  Если кому интересно, канеш. 

По вопросам стучите ваську 394705231 или на форумах amperka.ru и arduino.ru
