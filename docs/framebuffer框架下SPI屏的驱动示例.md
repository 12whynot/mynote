```
#include <linux/types.h>
#include <linux/kernel.h>
#include <linux/delay.h>
#include <linux/ide.h>
#include <linux/init.h>
#include <linux/module.h>
#include <linux/errno.h>
#include <linux/gpio.h>
#include <linux/cdev.h>
#include <linux/device.h>
#include <linux/of_gpio.h>
#include <linux/semaphore.h>
#include <linux/timer.h>
#include <linux/spi/spi.h>
#include <linux/of.h>
#include <linux/of_address.h>
#include <linux/of_gpio.h>
#include <linux/platform_device.h>
#include <asm/mach/map.h>
#include <asm/uaccess.h>
#include <asm/io.h>
#include <linux/fb.h>
#include <linux/dma-mapping.h>
#include <linux/sched.h>
#include <linux/wait.h>
#include <linux/fs.h> 
#include <linux/module.h> 
#include <linux/uaccess.h> 
#include <asm/mach/map.h>
 
#define DEVICE_NAME		"spi_st7796u"	/* 名字 */
 
/*
 * 自定义结构体spi_st7796u_lcd_dev
 */
struct spi_st7796u_lcd_dev {
	dev_t  devid;					/* 设备号 */
	struct cdev cdev;				/* cdev结构体 */
	struct class *class;			/* 类 */
	struct device *device;			/* 设备 */
	void   *private_data;     
	
	int    dc_gpio;
    int    rest_gpio;
};
 
static struct spi_st7796u_lcd_dev spi_st7796u_lcd;	
 
typedef struct{
    struct spi_device *spi;     //记录fb_info对象对应的spi设备对象
    struct task_struct *thread; //记录线程对象的地址，此线程专用于把显存数据发送到屏的驱动ic
}lcd_data_t;
 
#define SPI_SPEED 25000000
 
#define RED  	0xf800
#define GREEN	0x07e0
#define BLUE 	0x001f
#define WHITE	0xffff
#define BLACK	0x0000
#define YELLOW  0xFFE0
#define GRAY0   0xEF7D   		//灰色0 3165 00110 001011 00101
#define GRAY1   0x8410      	//灰色1      00000 000000 00000
#define GRAY2   0x4208      	//灰色2  1111111111011111
 
#define LCD_W 480
#define LCD_H 320
 
#define USE_HORIZONTAL  	 0	//顺时针旋转屏幕角度0-3
 
static int tft_lcdfb_setcolreg(unsigned int regno, unsigned int red,
			                    unsigned int green, unsigned int blue,
			                    unsigned int transp, struct fb_info *info);
 
struct fb_ops fops = {
    .owner = THIS_MODULE,
    .fb_setcolreg = tft_lcdfb_setcolreg,
    .fb_fillrect = cfb_fillrect,
    .fb_copyarea = cfb_copyarea,
    .fb_imageblit = cfb_imageblit,
};
 
void show_fb(struct fb_info *fbi, struct spi_device *spi);
 
 
/* 线程函数 */
int thread_func(void *data)
{
    struct fb_info *fbi = (struct fb_info *)data;
    lcd_data_t *ldata = fbi->par;   
 
    while(1)
    {
        if(kthread_should_stop())
            break;
        show_fb(fbi, ldata->spi);
    }
 
    return 0;
}
 
static u32 pseudo_palette[16];
static inline unsigned int chan_to_field(unsigned int chan, struct fb_bitfield *bf)
{
	chan &= 0xffff;
	chan >>= 16 - bf->length;
	return chan << bf->offset;
}
 
static int tft_lcdfb_setcolreg(unsigned int regno, unsigned int red,
			     unsigned int green, unsigned int blue,
			     unsigned int transp, struct fb_info *info)
{
	unsigned int val;
	
	if (regno > 16)
	{
		printk(KERN_ALERT "%s the regno is %d !!\n",__FUNCTION__, regno);
		return 1;
	}
		
 
	/* 用red,green,blue三原色构造出val  */
	val  = chan_to_field(red,	&info->var.red);
	val |= chan_to_field(green, &info->var.green);
	val |= chan_to_field(blue,	&info->var.blue);
	
	pseudo_palette[regno] = val;
	return 0;
}
 
/* 此函数在spi设备驱动的probe函数里被调用 */
struct fb_info * fb_init(struct spi_device *spi)
{
    struct fb_info *fbi;
    u8 *v_addr;
    dma_addr_t p_addr;
    lcd_data_t *ldata;
 
	spi_st7796u_lcd.device->coherent_dma_mask = ~0;
	v_addr = dma_alloc_coherent(spi_st7796u_lcd.device, LCD_W * LCD_H * 4, &p_addr, GFP_KERNEL);
    if(v_addr)
	{
		printk(KERN_ALERT "allocate CM at virtual address: 0x%p"
			"address: 0x%p size:%dKiB\r\n", v_addr,
			(void *)p_addr, LCD_W * LCD_H * 4 / SZ_1K);
	}
	else
	{
		printk(KERN_ALERT "no mem in CMA area\r\n");
	}
	fbi = framebuffer_alloc(sizeof(lcd_data_t), NULL);//额外分配lcd_data_t类型空间
    ldata = fbi->par;       //datal指针指向额外分配的空间
    ldata->spi = spi;
 
   	fbi->pseudo_palette = pseudo_palette;
	fbi->var.activate = FB_ACTIVATE_NOW;
 
    /* 设置fbi的变量 RGB这里需要注意长度和偏移*/
    fbi->var.xres = LCD_W;
    fbi->var.yres = LCD_H;
    fbi->var.xres_virtual = LCD_W;
    fbi->var.yres_virtual = LCD_H;
    fbi->var.bits_per_pixel = 32; // 屏是rgb565, 但QT程序只能支持32位.还需要在刷图时把32位的像素数据转换成rgb565
    fbi->var.red.offset = 16;
    fbi->var.red.length = 8;
    fbi->var.green.offset = 8;
    fbi->var.green.length = 8;
    fbi->var.blue.offset = 0;
    fbi->var.blue.length = 8;
 
    /* 设置fbi的常量 */
    strcpy(fbi->fix.id, "spi_st7796u");
    fbi->fix.smem_start = p_addr; //显存的物理地址
    fbi->fix.smem_len = LCD_W * LCD_H * 4;
    fbi->fix.type = FB_TYPE_PACKED_PIXELS;
    fbi->fix.visual = FB_VISUAL_TRUECOLOR;
    fbi->fix.line_length = LCD_W * 4;
 
    /* 其他重要的成员 */
    fbi->fbops = &fops;
    fbi->screen_base = v_addr;//显存虚拟地址
    fbi->screen_size = LCD_W * LCD_H * 4;//显存大小
 
    spi_set_drvdata(spi, fbi);  //spi成员关联了fbi
 
    register_framebuffer(fbi);  //注册初始化完成的fbi
    ldata->thread = kthread_run(thread_func, fbi, spi->modalias);
 
    return fbi;
}
 
/* 此函数在spi设备驱动remove时被调用 */
void fb_del(struct spi_device *spi)
{
    struct fb_info *fbi = spi_get_drvdata(spi); //得到之前存在spi设备中的fbi
    lcd_data_t *ldata = fbi->par;
 
    kthread_stop(ldata->thread); //让刷图线程退出
    unregister_framebuffer(fbi);
    dma_free_coherent(NULL, fbi->screen_size, fbi->screen_base, fbi->fix.smem_start);//把物理地址和虚拟地址放进去
    framebuffer_release(fbi);
}
 
static int LCD_WriteReg(struct spi_device *spi, unsigned char c)
{
	struct spi_message msg = {0};
	struct spi_transfer xfer = {0};
	unsigned char tx;
	int ret;
 
	gpio_set_value(spi_st7796u_lcd.dc_gpio, 0);/* 拉低发命令 */
 
	xfer.speed_hz = SPI_SPEED,
	xfer.tx_buf = &tx;		/* 发送数据缓存区 */
	xfer.bits_per_word = 8;	/* 一次传输8个bit */
	xfer.len = 1;			/* 传输长度为2个字节：寄存器地址+写入值 */
 
	tx = c;			/* 要写入的从机设备寄存器地址（举个例子，假设寄存器地址是一个字节） */
	spi_message_init(&msg);	/* 初始化spi_message */
	spi_message_add_tail(&xfer, &msg);	/* 将spi_transfer添加到spi_message */
	ret = spi_sync(spi, &msg);		/* 执行同步数据传输 */
	if (ret)
		return ret;
		
	return 0;
}
 
static int LCD_WriteReg_len(struct spi_device *spi, u8 *buf, unsigned int len)
{
	int ret;
 
    struct spi_transfer *t;
	struct spi_message m;
 
	t = kzalloc(sizeof(struct spi_transfer), GFP_KERNEL);	/* 申请内存 */
	
	gpio_set_value(spi_st7796u_lcd.dc_gpio, 0);/* 拉低发命令 */
 
	t->speed_hz = SPI_SPEED,
	t->tx_buf = buf;			/* 要发送的数据 */
	t->len = len;				/* 字节 */
	spi_message_init(&m);		/* 初始化spi_message */
	spi_message_add_tail(t, &m);/* 将spi_transfer添加到spi_message队列 */
	ret = spi_sync(spi, &m);	/* 同步发送 */
 
	kfree(t);					/* 释放内存 */
 
	return ret;
}
 
 
static int LCD_WriteData(struct spi_device *spi, unsigned char c)
{
	struct spi_message msg = {0};
	struct spi_transfer xfer = {0};
	unsigned char tx;
	int ret;
 
    gpio_set_value(spi_st7796u_lcd.dc_gpio, 1);    /* 拉高发数据 */
 
	xfer.speed_hz = SPI_SPEED,
	xfer.tx_buf = &tx;		/* 发送数据缓存区 */
	xfer.bits_per_word = 8;	/* 一次传输8个bit */
	xfer.len = 1;			/* 传输长度为2个字节：寄存器地址+写入值 */
 
	tx = c;			/* 要写入的从机设备寄存器地址（举个例子，假设寄存器地址是一个字节） */
	spi_message_init(&msg);	/* 初始化spi_message */
	spi_message_add_tail(&xfer, &msg);	/* 将spi_transfer添加到spi_message */
	ret = spi_sync(spi, &msg);		/* 执行同步数据传输 */
	if (ret)
		return ret;
		
	return 0;
}
 
static int LCD_WriteData_16bit(struct spi_device *spi, unsigned short dat)
{
	int ret;
	struct spi_message msg = {0};
	struct spi_transfer xfer = {0};
	unsigned char tx = dat>>8;;
    gpio_set_value(spi_st7796u_lcd.dc_gpio, 1);    /* 拉高发数据 */
 
	xfer.speed_hz = SPI_SPEED,
	xfer.tx_buf = &tx;		/* 发送数据缓存区 */
	xfer.bits_per_word = 8;	/* 一次传输8个bit */
	xfer.len = 1;			/* 传输长度为2个字节：寄存器地址+写入值 */
 
    spi_message_init(&msg);   /* 初始化spi_message */
    spi_message_add_tail(&xfer, &msg);    /* 将spi_transfer添加到spi_message队列 */
 
    ret = spi_sync(spi, &msg);        /* 使用同步发送 */
 
    tx = dat&0xFF;
    xfer.tx_buf = &tx;
    ret = spi_sync(spi, &msg);        /* 使用同步发送 */
	if (ret)
		return ret;
	
    return 0;
}
 
static int LCD_WriteData_len(struct spi_device *spi, u8 *buf, unsigned int len)
{
	int ret;
 
    struct spi_transfer *t;
	struct spi_message m;
 
	t = kzalloc(sizeof(struct spi_transfer), GFP_KERNEL);	/* 申请内存 */
	
    gpio_set_value(spi_st7796u_lcd.dc_gpio, 1);    /* 拉高发数据 */
 
	t->speed_hz = SPI_SPEED,
	t->tx_buf = buf;			/* 要发送的数据 */
	t->len = len;				/* 字节 */
	spi_message_init(&m);		/* 初始化spi_message */
	spi_message_add_tail(t, &m);/* 将spi_transfer添加到spi_message队列 */
	ret = spi_sync(spi, &m);	/* 同步发送 */
 
	kfree(t);					/* 释放内存 */
 
	return ret;
}
 
 
static void LCD_Set_Window(struct spi_device *spi,u16 xstart,u16 ystart,u16 xend,u16 yend)
{
    LCD_WriteReg(spi, 0x2A);
	LCD_WriteData_16bit(spi, xstart);
	LCD_WriteData_16bit(spi, xend);
 
    LCD_WriteReg(spi, 0x2B); 
    LCD_WriteData_16bit(spi, ystart);
    LCD_WriteData_16bit(spi, yend);  
 
    LCD_WriteReg(spi, 0x2C); //开始写入GRAM
}
 
/* framebuffer线程刷屏函数 fbi显存的数据给spi发送*/
void show_fb(struct fb_info *fbi, struct spi_device *spi)
{
	int x, y, i = 0;
    u32 k;
    u32 *p = (u32 *)(fbi->screen_base);
    u16 c;
    u8 *pp;
 
	/* 申请内存 */
	u8 *lcd_buffer = kzalloc(LCD_W * LCD_H * sizeof(u16), GFP_KERNEL);
 
    LCD_Set_Window(spi, 0,0,LCD_W-1,LCD_H-1);
    for (y = 0; y < fbi->var.yres; y++)
    {
        for (x = 0; x < fbi->var.xres; x++)
        {
            k = p[y*fbi->var.xres+x];//取出一个像素点的32位数据
            // rgb8888 --> rgb565       
            pp = (u8 *)&k;
            c = pp[0] >> 3; //蓝色
            c |= (pp[1]>>2)<<5; //绿色
            c |= (pp[2]>>3)<<11; //红色
			lcd_buffer[i++] = (c>>8)&0xff;
			lcd_buffer[i++] = c&0xff;
        }
    }
	LCD_WriteData_len(spi, lcd_buffer, LCD_W * LCD_H * sizeof(u16));
 
	/* 释放内存 */
    kfree(lcd_buffer);
}
 
void LCD_Disp_Pic(struct spi_device *spi)
{
	/* 申请内存 */
	u8 *lcd_buffer = kzalloc(LCD_W * LCD_H * sizeof(u16), GFP_KERNEL);
 
    /* 设置屏幕和显示部分 */
    LCD_Set_Window(spi, 0, 0, LCD_W-1, LCD_H-1);
	/* 对内存赋值 */
	memset(lcd_buffer, BLUE, LCD_W * LCD_H * sizeof(u16));
	/* 连续写 */
	LCD_WriteData_len(spi, lcd_buffer, LCD_W * LCD_H * sizeof(u16));
 
	mdelay(500);
 
	/* 释放内存 */
    kfree(lcd_buffer);
 
    printk(KERN_ALERT "LCD display picture ok!\r\n");
}
 
static void lcd_reg_init(struct spi_device *spi)
{
    printk(KERN_ALERT "This is lcd reg init func!\r\n");
 
    gpio_set_value(spi_st7796u_lcd.rest_gpio, 0);
    mdelay(100);
    gpio_set_value(spi_st7796u_lcd.rest_gpio, 1);
    mdelay(50);
    /******************************/
	LCD_WriteReg(spi,0x11);     
 
	mdelay(120);                //Delay 120ms
 
	LCD_WriteReg(spi,0x36);     // Memory Data Access Control MY,MX~~
	LCD_WriteData(spi,0x48);   
 
	LCD_WriteReg(spi,0x3A);     
	LCD_WriteData(spi,0x55);   //LCD_WriteData(0x66);
 
	LCD_WriteReg(spi,0xF0);     // Command Set Control
	LCD_WriteData(spi,0xC3);   
 
	LCD_WriteReg(spi,0xF0);     
	LCD_WriteData(spi,0x96);   
 
	LCD_WriteReg(spi,0xB4);     
	LCD_WriteData(spi,0x01);   
 
	LCD_WriteReg(spi,0xB7);     
	LCD_WriteData(spi,0xC6);   
 
 
	LCD_WriteReg(spi,0xC0);     
	LCD_WriteData(spi,0x80);   
	LCD_WriteData(spi,0x45);   
 
	LCD_WriteReg(spi,0xC1);     
	LCD_WriteData(spi,0x13);   //18  //00
 
	LCD_WriteReg(spi,0xC2);     
	LCD_WriteData(spi,0xA7);   
 
	LCD_WriteReg(spi,0xC5);     
	LCD_WriteData(spi,0x0A);   
 
	LCD_WriteReg(spi,0xE8);     
	LCD_WriteData(spi,0x40);
	LCD_WriteData(spi,0x8A);
	LCD_WriteData(spi,0x00);
	LCD_WriteData(spi,0x00);
	LCD_WriteData(spi,0x29);
	LCD_WriteData(spi,0x19);
	LCD_WriteData(spi,0xA5);
	LCD_WriteData(spi,0x33);
 
	LCD_WriteReg(spi,0xE0);
	LCD_WriteData(spi,0xD0);
	LCD_WriteData(spi,0x08);
	LCD_WriteData(spi,0x0F);
	LCD_WriteData(spi,0x06);
	LCD_WriteData(spi,0x06);
	LCD_WriteData(spi,0x33);
	LCD_WriteData(spi,0x30);
	LCD_WriteData(spi,0x33);
	LCD_WriteData(spi,0x47);
	LCD_WriteData(spi,0x17);
	LCD_WriteData(spi,0x13);
	LCD_WriteData(spi,0x13);
	LCD_WriteData(spi,0x2B);
	LCD_WriteData(spi,0x31);
 
	LCD_WriteReg(spi,0xE1);
	LCD_WriteData(spi,0xD0);
	LCD_WriteData(spi,0x0A);
	LCD_WriteData(spi,0x11);
	LCD_WriteData(spi,0x0B);
	LCD_WriteData(spi,0x09);
	LCD_WriteData(spi,0x07);
	LCD_WriteData(spi,0x2F);
	LCD_WriteData(spi,0x33);
	LCD_WriteData(spi,0x47);
	LCD_WriteData(spi,0x38);
	LCD_WriteData(spi,0x15);
	LCD_WriteData(spi,0x16);
	LCD_WriteData(spi,0x2C);
	LCD_WriteData(spi,0x32);
	 
 
	LCD_WriteReg(spi,0xF0);     
	LCD_WriteData(spi,0x3C);   
 
	LCD_WriteReg(spi,0xF0);     
	LCD_WriteData(spi,0x69);   
 
	mdelay(120);                
 
	LCD_WriteReg(spi,0x21);     
	LCD_WriteReg(spi,0x29); 
 
	/* 横向显示 */
	LCD_WriteReg(spi,0x36);  
	LCD_WriteData(spi,0x28);	  
	
    printk(KERN_ALERT "LCD init ok!\r\n");
 
    LCD_Disp_Pic(spi);
}
 
static int lcd_gpio_init(struct spi_device *spi)
{
    int ret;
    int flag1,flag2;
 
    /* 获取GPIO */
    spi_st7796u_lcd.dc_gpio = of_get_named_gpio_flags(spi->dev.of_node, "dc-gpios", 0, (enum of_gpio_flags *)&flag1);
    ret = gpio_is_valid(spi_st7796u_lcd.dc_gpio);
    if(ret < 0)
    {
        printk(KERN_ALERT "get dc gpio failed !\r\n");
        return -EINVAL;
    }
    ret = gpio_request(spi_st7796u_lcd.dc_gpio,"dc-gpios");
    if(ret < 0)
    {
        printk(KERN_ALERT "request gpio dc failed !\r\n");
        gpio_free(spi_st7796u_lcd.dc_gpio);
    }
    printk(KERN_ALERT "dc gpio num is %d !\r\n",spi_st7796u_lcd.dc_gpio);
 
    spi_st7796u_lcd.rest_gpio = of_get_named_gpio_flags(spi->dev.of_node, "reset-gpios", 0, (enum of_gpio_flags *)&flag2);
    ret = gpio_is_valid(spi_st7796u_lcd.rest_gpio);
    if(ret < 0)
    {
        printk(KERN_ALERT "get rest gpio failed !\r\n");
        return -EINVAL;
    }
    ret = gpio_request(spi_st7796u_lcd.rest_gpio,"reset-gpios");
    if(ret < 0)
    {
        printk(KERN_ALERT "request gpio rest failed !\r\n");
        gpio_free(spi_st7796u_lcd.rest_gpio);
    }
    printk(KERN_ALERT "rest gpio num is %d !\r\n",spi_st7796u_lcd.rest_gpio);
 
    /* 设置方向 */
	gpio_direction_output(spi_st7796u_lcd.dc_gpio,0);
	gpio_direction_output(spi_st7796u_lcd.rest_gpio,0);
 
    return ret;
}
 
/*
 * @description			: 打开设备
 * @param – inode		: 传递给驱动的inode
 * @param - filp		: 设备文件，file结构体有个叫做private_data的成员变量
 * 						  一般在open的时候将private_data指向设备结构体。
 * @return				: 0 成功;其他 失败
 */
static int spi_st7796u_lcd_open(struct inode *inode, struct file *filp)
{
	filp->private_data = &spi_st7796u_lcd;
	return 0;
}
 
/*
 * @description			: 从设备读取数据 
 * @param – filp		: 要打开的设备文件(文件描述符)
 * @param - buf			: 返回给用户空间的数据缓冲区
 * @param - cnt			: 要读取的数据长度
 * @param – off			: 相对于文件首地址的偏移
 * @return				: 读取的字节数，如果为负值，表示读取失败
 */
static ssize_t spi_st7796u_lcd_read(struct file *filp, char __user *buf,
			size_t cnt, loff_t *off)
{
	return 0;
}
 
/*
 * @description			: 向设备写数据 
 * @param – filp		: 设备文件，表示打开的文件描述符
 * @param - buf			: 要写给设备写入的数据
 * @param - cnt			: 要写入的数据长度
 * @param - offt		: 相对于文件首地址的偏移
 * @return				: 写入的字节数，如果为负值，表示写入失败
 */
static ssize_t spi_st7796u_lcd_write(struct file *filp, const char __user *buf,
			size_t cnt, loff_t *offt)
{
	return 0;
}
 
 
static ssize_t spi_st7796u_lcd_write_reg(struct spi_device *spi, unsigned char c)
{
	//struct spi_st7796u_lcd_dev *dev = spi_dev->private_data;
	struct spi_message msg = {0};
	struct spi_transfer xfer = {0};
	unsigned char tx;
	int ret;
 
	xfer.tx_buf = &tx;		/* 发送数据缓存区 */
	xfer.bits_per_word = 8;	/* 一次传输8个bit */
	xfer.len = 1;			/* 传输长度为2个字节：寄存器地址+写入值 */
 
	tx = c;			/* 要写入的从机设备寄存器地址（举个例子，假设寄存器地址是一个字节） */
	spi_message_init(&msg);	/* 初始化spi_message */
	spi_message_add_tail(&xfer, &msg);	/* 将spi_transfer添加到spi_message */
	//ret = spi_sync(dev->spi, &msg);		/* 执行同步数据传输 */
	ret = spi_sync(spi, &msg);		/* 执行同步数据传输 */
	if (ret)
		return ret;
	
	printk(KERN_ALERT "spi test\r\n");
		
	return 0;
}
 
/*
 * @description			: 关闭/释放设备
 * @param - filp		: 要关闭的设备文件(文件描述符)
 * @return				: 0 成功;其他 失败
 */
static int spi_st7796u_lcd_release(struct inode *inode, struct file *filp)
{
	return 0;
}
 
/*
 * file_operations结构体变量
 */
static const struct file_operations spi_st7796u_lcd_ops = {
	.owner = THIS_MODULE,
	.open = spi_st7796u_lcd_open,
	.read = spi_st7796u_lcd_read,
	.write = spi_st7796u_lcd_write,
	.release = spi_st7796u_lcd_release,
};
 
static int spi_st7796u_lcd_init(struct spi_st7796u_lcd_dev *dev)
{
	/* 对设备进行初始化操作 */
	/* 在这个函数中对SPI从机设备进行相关初始化操作 */
	/* ...... */
	return 0;
}
 
static int spi_st7796u_lcd_probe(struct spi_device *spi)
{
	int ret;
	struct fb_info *fbi;
 
    printk(KERN_ALERT "spi_lcd_probe ok!\r\n");
 
	/* 初始化虚拟设备 */
	ret = spi_st7796u_lcd_init(&spi_st7796u_lcd);
	if (ret)
		return ret;
 
	/* 申请设备号 */
	ret = alloc_chrdev_region(&spi_st7796u_lcd.devid, 0, 1, DEVICE_NAME);
	if (ret)
		return ret;
 
	/* 初始化字符设备cdev */
	spi_st7796u_lcd.cdev.owner = THIS_MODULE;
	cdev_init(&spi_st7796u_lcd.cdev, &spi_st7796u_lcd_ops);
 
	/* 添加cdev */
	ret = cdev_add(&spi_st7796u_lcd.cdev, spi_st7796u_lcd.devid, 1);
	if (ret)
		goto out1;
 
	/* 创建类class */
	spi_st7796u_lcd.class = class_create(THIS_MODULE, DEVICE_NAME);
	if (IS_ERR(spi_st7796u_lcd.class)) {
		ret = PTR_ERR(spi_st7796u_lcd.class);
		goto out2;
	}
 
	/* 创建设备 */
	spi_st7796u_lcd.device = device_create(spi_st7796u_lcd.class, &spi->dev,
				spi_st7796u_lcd.devid, NULL, DEVICE_NAME);
	if (IS_ERR(spi_st7796u_lcd.device)) {
		ret = PTR_ERR(spi_st7796u_lcd.device);
		goto out3;
	}
 
    /*初始化spi_device */
    spi_st7796u_lcd.private_data = spi;
 
	/* 初始化lcd屏的dc rest管脚 */
    lcd_gpio_init(spi);
	
	/* 内部寄存器初始化 */
    lcd_reg_init(spi);
 
	/* 初始化fb设备 */
    fbi = fb_init(spi);
	
	return 0;
 
out3:
	class_destroy(spi_st7796u_lcd.class);
 
out2:
	cdev_del(&spi_st7796u_lcd.cdev);
 
out1:
	unregister_chrdev_region(spi_st7796u_lcd.devid, 1);
 
	return ret;
}
 
static int spi_st7796u_lcd_remove(struct spi_device *spi)
{
	struct spi_st7796u_lcd_dev *dev = spi_get_drvdata(spi);
 
    /* fb设备回收 */
    fb_del(spi);
 
	/* 注销设备 */
	device_destroy(dev->class, dev->devid);
 
	/* 注销类 */
	class_destroy(dev->class);
 
	/* 删除cdev */
	cdev_del(&dev->cdev);
 
	/* 注销设备号 */
	unregister_chrdev_region(dev->devid, 1);
 
	return 0;
}
 
/* 匹配列表 */
static const struct of_device_id spi_st7796u_lcd_of_match[] = {
	{ .compatible = "sitronix,st7796" },
	{ /* Sentinel */ }
};
 
/* SPI总线下的设备驱动结构体变量 */
static struct spi_driver spi_st7796u_lcd_driver = {
	.driver = {
		.name			= "st7796u",
		.of_match_table	= spi_st7796u_lcd_of_match,
	},
	.probe		= spi_st7796u_lcd_probe,		// probe函数
	.remove		= spi_st7796u_lcd_remove,		// remove函数
};
 
module_spi_driver(spi_st7796u_lcd_driver);
 
MODULE_AUTHOR("WQH");
MODULE_DESCRIPTION("st7796u_spi_lcd");
MODULE_LICENSE("GPL");
```