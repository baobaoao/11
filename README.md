lcd1602.c

#include<reg51.h>

#include<intrins.h>

#define INT8U unsigned char

#define INT16U unsigned int

sbit RS=P2^0;

sbit RW=P2^1;

sbit EN=P2^2;

void delay_ms(INT16U x)

{

INT8U i;

while(x--)

{

for(i=0;i<120;i++);

}

}

void Busy_Wait()

{

INT8U LCD_Status;

do

{

P0=0xFF;

EN=0;RS=0;RW=1;

EN=1; LCD_Status=P0;

EN=0;

}

while(LCD_Status&0x80);//1000 0000

}

void Write_LCD_Command(INT8U cmd)



{

 Busy_Wait();

 EN=0;RS=0;RW=0;

 P0=cmd;

 EN=1;_nop_();EN=0;



}

void Write_LCD_Data(INT8U data1)

{

 Busy_Wait();

 EN=0;RS=1;RW=0;

 P0=data1;

 EN=1;_nop_();EN=0;



}

void LCD_String(INT8U r,INT8U c,INT8U *str)

{

	INT8U i=0;

	INT8U code DDRAM[]={0x80,0xC0};

	Write_LCD_Command(DDRAM[r]|c);

	for(i=0;i<16&&str[i];i++)

	{

	Write_LCD_Data(str[i]);

	}

	for(;i<16;i++)

	Write_LCD_Data(' ');

}

void LCD_Initialize()

{

Write_LCD_Command(0x38);delay_ms(1);//置功能位，8位，双行，每一个字符占5*7

Write_LCD_Command(0x01);delay_ms(1);//清屏 

Write_LCD_Command(0x06);delay_ms(1);//字符进入模式，屏幕不动，字符后移

Write_LCD_Command(0x0C);delay_ms(1);//开显示，关光标

}













DSB18B20.c



#include<reg51.h>//register

#include<intrins.h>//_nop_() _crol_()

#define INT8U unsigned char

#define INT16U unsigned int

#define delay4us();	{_nop_();_nop_();_nop_();_nop_();}

extern void delay_ms(INT16U x);

sbit DQ=P2^3;

INT8U Temp_Value[]={0x00,0x00};	//温度数据的低8位与高8位

/*void delay_ms(INT16U x)

{

INT8U i;

while(x--)

{

for(i=0;i<120;i++);

}

}

*/

void delayx(INT16U x)

{

while(x--);

}

/***********AT89C51的主频一定要设为11.0592MHz*****/

INT8U Init_DS18B20()//初始化和检测DS18B20

{

INT8U DQ_status;

DQ=1; delayx(8);//delay77us

DQ=0; delayx(90);//>delay480us

DQ=1; delayx(5);//>delay15us

DQ_status=DQ; delayx(90);

return DQ_status;

}





INT8U Readonebyte()//读取DS18B20中的1个字节的数据

 {

 INT8U i,datatemp=0x00;

 for(i=0x01;i!=0x00;i<<=1)//100000000

 {

 DQ=0;_nop_();  

 DQ=1;_nop_();

 if(DQ) datatemp=datatemp|i; 

 delayx(8);

 }

 return datatemp;



 }

 void Writeonebyte(INT8U dat)//dat>>1 00001010 1CY ACC

 { INT8U i;

 for(i=0;i<8;i++)

 {

 DQ=1;_nop_(); 

 DQ=0;_nop_();

 dat=dat>>1;  //PSW

 DQ=CY;

 delayx(8); 

 

 }



 }

 INT8U Read_temperature()

 {

 if(Init_DS18B20()==1) return 0;

 else

 {

  Writeonebyte(0xCC);//跳检序列号

  Writeonebyte(0x44);//启动转换

  Init_DS18B20(); 

  Writeonebyte(0xCC);//跳检序列号 因为只有一个DS18B20

  Writeonebyte(0xBE);//读温度寄存器里的第0，1字节

 Temp_Value[0]=Readonebyte(); 

 Temp_Value[1]=Readonebyte();

 return 1;



 }

 }



mainc.c



#include<reg51.h>

#include<intrins.h>

#include<string.h>

#include<stdio.h>



#define INT8U unsigned char

#define INT8 signed char

#define INT16U unsigned int	

#define Max_Char 11



extern void LCD_String(INT8U r,INT8U c,INT8U *str);

extern void LCD_Initialize();

extern void delay_ms(INT16U x);

extern INT8U Init_DS18B20();

extern INT8U Read_temperature();

extern INT8U Temp_Value[];  //储存温度采集数据，采集的16位温度数据放在这里

float f_temp=35.0;		//浮点型的温度数据

char temp_disp_buff[17];	//温度的显示字符

volatile INT8U recv_buff[Max_Char+1];

volatile INT8U Buf_index=0;

volatile INT8U recv_ok=0;





sbit K1=P1^5;		//通风电机开关

sbit K2=P1^6;		//采光电机开关

sbit K3=P1^7;		//水泵电机开关

sbit F_in1=P1^0;		//通风电机驱动线1	

sbit F_in2=P1^1;		//通风电机驱动线2

sbit F_in3=P1^2;		//采光电机驱动线1

sbit F_in4=P1^3;		//采光电机驱动线2

sbit LED1=P2^5;			//通风电机运行显示灯

sbit LED2=P2^6;			//采光电机运行显示灯

sbit LED3=P2^7;			//水泵电机运行显示灯

sbit Delaypump=P2^4;		//水泵控制线

	

void putstr(char *s)	

{

	INT8U i=0;

	while(s[i])

	{

		SBUF=s[i++];

		while(TI==0);

		TI=0;

	}

}



void serial_INT_ISR() interrupt 4  // 0 INT0 , 1 T0INT , 2  INT1 , 3 T1INT ,4 SPIN 

{

	static INT8U i=0;

	INT8U c;

	if(RI==0)

		return;

	RI=0;

	c=SBUF;

	if(c=='$')

	{

		i=0;

		return;

	}

	// serialPort1.WriteLine("$LIGHT OPEN") 上位机最后会多发送2个: 0x0D 回车  0x0A 换行

	if(c==0x0D)

		return;

	if(c==0x0A)

		recv_ok=1;

	else

	{

		recv_buff[i]=c;

		recv_buff[++i]='\0';

		if(i==Max_Char)

			i=0;

	}

}





void INT_ISR() interrupt 0  //ISR---- Interrupt Service

{

	if(K1==0)

	{

		delay_ms(10);

		if(K1==0)

		{

			if(LED1==1)

			{

				F_in1=1;

				F_in2=1;

				LED1=0;

			}

			else

			{

				F_in1=1;

				F_in2=0;

				LED1=1;

			}

		}

	}

	

	if(K2==0)

	{

		delay_ms(10);

		if(K2==0)

		{

			if(LED2==1)

			{

				F_in3=1;

				F_in4=1;

				LED2=0;

			}

			else

			{

				F_in3=1;

				F_in4=0;

				LED2=1;

			}

		}

	}

	

	if(K3==0)

	{

		delay_ms(10);  //延时的作用是确定是否真的按下按钮，不是误操作

		if(K3==0)

		{

			if(LED3==1)

			{

				Delaypump=1;

				LED3=0;

			}

			else

			{

				Delaypump=0;

				LED3=1;

			}

		}

	}

}



void main()

{

	

	f_temp=0.0;

	LED1=LED2=LED3=0;

	SCON=0x50;  //0101 0000 串口方式1，允许接收REN=1

	TMOD=0x20;  //0010 0000 选择T1 定时方式为2

	PCON=0x00;  //不倍增 SMOD=0

	TH1=0xFD;   //通过波特率软件计算

	TL1=0xFD;

	EA=1;  //开总中断

	ES=1;  //开串口中断

	EX0=1;  //IE EA 0 0 ES ET1 ET0 EX0

	IT0=1;  //IT0=0,0触发有效； IT0=1,下降沿触发有效  按钮按一下有一段时间会处于电平一直处于0，如果0触发有效，会多次触发

	TR1=1;  //开启定时器计数

	//HTH1=TL1=0xFD;

	

	LCD_Initialize();

	LCD_String(0,0,"GreenHouse Test ");

	LCD_String(1,0,"Temp:  ");

	Read_temperature();

	delay_ms(800);

	

	

	while(1)

	{

		/*F_in1=1;

		F_in2=0;

		F_in3=1;

		F_in4=0;*/

		

		if(Read_temperature())

		{

			f_temp=(int)(Temp_Value[1]<<8|Temp_Value[0])*0.0625;

			sprintf(temp_disp_buff,"%5.1f",f_temp);

			putstr(strcat(temp_disp_buff,"\r\n"));

			strcat(temp_disp_buff,"\xDF\x43");

			LCD_String(1,7,temp_disp_buff);

			delay_ms(50);

		}

		if(recv_ok)

		{

			recv_ok=0;

			if(strcmp(recv_buff,"WIND_OPEN")==0)

			{

				F_in1=1;

				F_in2=0;

				LED1=1;

			}

			else if(strcmp(recv_buff,"LIGHT_OPEN")==0)

			{

				F_in3=1;

				F_in4=0;

				LED2=1;

			}

			else if(strcmp(recv_buff,"PUMP_OPEN")==0)

			{

				Delaypump=0;

				LED3=0;

			}

			else if(strcmp(recv_buff,"WIND_CLOSE")==0)

			{

				F_in1=1;

				F_in2=1;

				LED1=0;

			}

			else if(strcmp(recv_buff,"LIGHT_CLOSE")==0)

			{

				F_in3=1;

				F_in4=1;

				LED2=0;

			}

			else if(strcmp(recv_buff,"PUMP_CLOSE")==0)

			{

				Delaypump=1;

				LED3=0;

			}

		}

	}

}
