# 1、 框架模板
```
static ssize_t xxx_write(struct file *filp, const char __user*buf, size_t cnt,  loff_t*off_t)
{
	//实现具体的操作
}
static int xxx_open(struct inode*node, struct file*filp)
{
	//具体操作
}
//文件操作函数结构体，Linux通过此结构体访问驱动程序
static struct file_operations op={
	.owner = THIS_MODULE,
	.open  = xxx_open,
	.write = xxx_wrote,
	//除最基本的读写外，还有其它操作。如需要，须事先实现
};



//模块初始化函数
static int __init XXX_init(void)
{
	//读取设备树，获取对应信息和操作
	//通用
	//1、申请设备号
	alloc_chrdev_region();
	//Major = MAJOR(填入对应的dev_t设备号);  //获取主设备号
	//Minor = MINOR(填入对应的dev_t设备号);  //获取次设备号

	//2、将struct cdev类型的结构体变量和file_operations结构体进行绑定
	cdev_init(struct cedv*cdev,const struct file_operation*fops);
	
	//3、向内核添加一个驱动，注册驱动
	cdev_add();
	//4、创建类
	class_create();
	//5、创建设备
	device_create();
}

//模块退出函数
static void __exit xxx_exit()
{
	cdev_del();

    unregister_chrdev_region();

    device_destroy();

    class_destroy();
    
}

module_init(XXX_init);
module_exit(xxx_exit);
MODULE_AUTHOR("author_name");
MODULE_LICEENSE("GPL");
```  

# 2、常用API函数解析
1. cdev_init
2. cdev_add
3. cdev_del
4. alloc_chrdev_region
5. unregister_chrdev_region
6. class_create
7. class_destroy
8. device_create
9. device_destroy
10. module_init
11. module_exit
12. struct file_operations
# 3、运行流程
	