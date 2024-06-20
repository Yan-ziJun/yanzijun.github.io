# 基于51单片机设计的交通信号灯
## 一、设计要求
    利用单片机的中断、定时器等功能，将定时器、中断的知识应用到交通信号灯控制系统中。要实现的功能是：正常交通信号灯模式为绿灯亮灯时间为30秒，黄灯亮灯时间为5秒，红灯亮灯时间为35秒，十字路口四个方向分别利用数码管显示倒计时数字，南北方向与东西方向交替执行。加入按键控制，按键1（K2）按下一次，实现倒计时加一，可重复执行；按键2（K3）按下一次，实现倒计时减一，可重复执行；按键3（K1）按下一次，实现倒计时设置确定和恢复正常交通信号灯模式；按键4（K4）按下，实现南北方向交通管制模式；按键5（K5）按下，实现东西方向交通管制模式；按键6（K6）按下，实现交警管制交通模式；按键7（K7）按下一次，实现可定时夜间交通模式；按键8（K8）按下一次，实现可定时优先主干道通行模式。按下K7和K8按键各一次，实现夜间至早高峰结束无人托管模式。
## 二、设计思路
### （一）正常交通信号灯模式设计
#### 1、倒计时数字显示设计
    在硬件电路方面，数码管采用4个两位七段共阴数码管，段码输入端与单片机输出段码端对应连接，公共端与单片机输出数码管动态显示信号端对应连接。
    
![image](https://github.com/Yan-ziJun/yanzijun.github.io/assets/162871983/6f81ac05-89a5-4415-a094-d1374efaf51c)

     在程序设计方面，定义一个全局变量 djs 作为倒计时数，将变量除以 10 得到数码管十位数字，变量取 10 的余得到数码管的个位数字，数码管的具体实现，采用数组存放0-9共阴极数字段码，我采取 number[] = {0-9的段码}; 刚好 number[数字] = 数字的段码；两个数字的依次显示（延时1ms）,数码管动态显示得以实现数码管倒计时数字的显示。
```
int number[] = {0x3f,0x06,0x5b,0x4f,0x66,0x6d,0x7d,0x07,0x7f,0x6f};
void smg_display()		 
{	
	int b1,b2;        
 	b1 = djs/10;        
	b2 = djs%10;     
	
	P0 = number[b1];    
        smg1 = 0;
        delay1ms();       
        smg1 = 1;

	P0 = number[b2];     
	smg2 = 0;
	delay1ms();       
	smg2 = 1;
}
```
#### 2、红绿灯显示设计
    在硬件电路方面，使用发光颜色为绿色、黄色、红色的LED作为红绿灯显示载体。担心单片机输出电流不能使 LED 发挥出最好的效果，所以采用 LED 低电平点亮。

![image](https://github.com/Yan-ziJun/yanzijun.github.io/assets/162871983/213ece07-cfdc-49b0-966b-4c99c46bf998)

    在程序设计方面，定义两个方向的红绿黄灯函数 EW_display()、SN_display() 通过一个标志位 led_flag 来改变红绿灯方向的来回切换。变换时间通过定时器的定时功能实现，设置定时器T1为1ms溢出一次，让变量 timer_1s 每 1ms 增加一次，增加 1000 次刚好为 1s ,作为倒计时秒计时，当 timer_1s 等于 1000 时，djs 减一，实现倒计时功能。让变量 timer_500ms 每 1ms 增加一次，增加 500 次刚好为 0.5s ,作为黄灯闪烁的时间，当 timer_500ms 等于 500 时，标志位 star_flag 取反，实现对黄灯的点亮和熄灭，实现黄灯的闪烁效果，每 0.5s 闪烁一次。（在定时器T1中断服务函数中实现）
```
int timer_1s    = 0;         //用于定时器计数变量
int djs         =35;		 //倒计时，不区分主次干道，一视同仁
bit led_flag    = 0;         //决定哪路先通行 0 南北  1 东西 详见main函数
int timer_500ms = 0;		 //用于定时器计数变量
bit star_flag   = 0;		 //黄灯闪烁效果，此位用来高低电平变换来实现黄灯闪烁的效果

void Timer1Init(void)		//1毫秒@12.000MHz
{
	TMOD = 0x10;
        TL1 = 0x18;		//设置定时初值
	TH1 = 0xFC;		//设置定时初值 
	TF1 = 0;		//清除TF1标志
	TR1 = 1;		//定时器1开始计时
	ET1 = 1;        //开定时器1中断
	EA = 1;         //开总中断
}
void timer1() interrupt 3 	
{
	TF1=0;
	TL1 = 0x18;		
	TH1 = 0xFC;		         
	timer_1s++;    
	timer_500ms++;
	if(timer_1s==1000)  
	{ 
		timer_1s=0;       
		djs--;                
		if(djs<0)
		{
			djs=35;             
			led_flag=~led_flag;        
		}
	}
	if(timer_500ms==500)  
	{
		timer_500ms=0;        
		star_flag=~star_flag;         
	}
}
```
### （二）按键功能设计
    在硬件电路方面，（下文按键使用 K 表示），K1实现倒计时设置确定和恢复正常交通信号灯模式，K2实现倒计时加一,K3实现倒计时减一，K4实现南北方向交通管制模式，K5实现东西方向交通管制模式，K6实现交警交通管制模式，K7实现可定时夜间交通模式， K8实现可定时优先主干道通行模式。K1、K2、K3、K7、K8采用轻触开关，K4、K5、K6采用自锁开关。

![image](https://github.com/Yan-ziJun/yanzijun.github.io/assets/162871983/7090daa3-20b3-44d3-96c0-2feb00c33e53)

    在程序设计方面，K1、K2、K3、K4、K5、K6都是在主函数内部函数触发实现的，K7、K8是触发外部中断来实现的。以下为各按键功能实现的程序设计：

```
//按键
void key()
{
    if(k1!=1)     //确定键（开启定时器）   
    {
		TR1=1;
	}
	if(k2!=1)      //加键  
	{
		smg_display();  //调用数码管显示函数     
		if(k2!=1)       //消抖，再次检测按键
		{
			TR1=0;      //关闭定时器
			djs++;      //倒计时变量加一
			if(djs>=99) //设置倒计时最大值
			{
				djs=99;    
			}       
			do          //防止按键抖动
			{
				smg_display();   
			}
			while(k2!=1);
		}
	}
	if(k3!=1)        //减键
	{
		smg_display();   //调用数码管显示函数 
		if(k3!=1)        //消抖，再次检测按键
		{
			TR1=0;       //关闭定时器
			djs--;       //倒计时变量减一
			if(djs<=0)   //设置倒计时最小值
			{
				djs=0;    
			}
			do           //防止按键抖动
			{		
			 smg_display();   
			}
			while(k3!=1);
		}
	}
	if(k4!=1)		//管制南北,东西通行
	{
		smg_display2(); //数码管显示特定样式
		TR1=0;          //关闭定时器
		EW_go();        //东西通行
		do              //防止抖动
		{
			smg_display2();
		}
		while(k4!=1);
	}
	if(k5!=1)		//管制东西，南北通行
	{
		smg_display2(); //数码管显示特定样式
		TR1=0;          //关闭定时器
		SN_go();        //南北通行
		do              //防止抖动
		{
			smg_display2();
		}
		while(k5!=1);
	}
	if(k6!=1)		//交警管制
	{
		smg_display2(); //数码管显示特定样式
		TR1=0;          //关闭定时器
		light_red();    //四个方向都为红灯
		do              //防止抖动
		{
			smg_display2();
		}
		while(k6!=1);
	}

}

void Init0() interrupt 0  //夜间模式
{
	unsigned int i;           
	unsigned long int j = 10;  //0-4294967295
	//1 hour       2    3     4      5    6       7     8     9     10      11    12
	//60*60=3600 7200 10800 14400 18000  21600  25200 28800 32400  36000  39600 43200 
	//  13    14    15    16    17   18    19     20    21    22    23     24
	//46800 50400 54000 57600 61200 64800 68400 72000 75600 79200  82800  86400  
	smg_out();
	while(j)
	{
		for(i=0;i<2;i++)
		{
			SN_green=1;       
			SN_red=1;         
			SN_yellow=i;	    	   
			EW_green=1;      
			EW_red=1;        
			EW_yellow=i;
			delay500ms();
		}
		j--;
	}
}

void Init1() interrupt 2   //主干道与非主干道优先级模式
{
	unsigned int i,f,k=5;
	unsigned long int t=2;  //T = (主干道djs + 非主干道djs)*t/2
	                        //t = 2*T/(主干道djs + 非主干道djs)
	smg_display();
	while(t)
	{
		for(i=0;i<2;i++)
		{
			if(i==0)
			{
				djs = 15;
				do
				{
					for(f=0;f<500;f++)
					{
						light_out();           
						SN_green=0;    
						EW_red=0;
						smg_display();
					}
					djs--;
					smg_display();
					if(djs==5)
					{
						while(k)
						{
							for(f=0;f<500;f++)
							{
								light_out();						 
								SN_green=1;
								SN_yellow = 0;
								EW_red=0;
								smg_display();
							}
							djs--;
							smg_display();
							k--;
						}
						k=5;
					}
					
				}
				while(djs>0);
				if(djs==0)
					{
						for(f=0;f<500;f++)
						{
							light_out();						 
							SN_green=1;
							SN_yellow = 0;
							EW_red=0;
							smg_display();
						}
					}
			}
			if(i==1)
			{
				djs = 10;
				do
				{
					for(f=0;f<500;f++)
					{
						smg_display();
						light_out();           
						EW_green=0;    
						SN_red=0;
					}
					djs--;
					if(djs==5)
					{
						while(k)
						{
							for(f=0;f<500;f++)
							{
								light_out();						 
								EW_green=1;
								EW_yellow = 0;
								SN_red=0;
								smg_display();
							}
							djs--;
							smg_display();
							k--;
						}
						k=5;
					}
				}
				while(djs>0);
				if(djs==0)
					{
						for(f=0;f<500;f++)
						{
							light_out();						 
							EW_green=1;
							EW_yellow = 0;
							SN_red=0;
							smg_display();
						}
					}
			}
		}
		t--;
	}
}
```
