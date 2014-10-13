---
layout: post
title: Python with as用法
category: "Python基础"
---
###python里的with ... as...语句作用###
 	with open('path',a) as file:
 				fileoperation
不用考虑对file的close操作，因为在执行完fileoperation后，会自动执行close。

还有用在对类的对象的操作上也很方便
 
 	with test() as a:
			
			.....
其实在a.fun之前会自动调用class里__enter__函数，执行完with下面的block后，会执行__exit__函数

    	class test:
		def __enter__(self):
      		print("enter")
       		return 1
		def __exit__(self,*args):
       		print("exit")
       		return True
	
###Python中sys.path.append()###
这个方法的作用类似于c语言里的#include头文件，把某一路径下的所有文件加入到系统路径中来，这样就可以在代码中直接调用包含的类或者函数等
