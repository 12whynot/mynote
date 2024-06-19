1. 预处理  
		eg: `gcc -E main.c -o main .i`
1. 编译  
		eg: `gcc -S main.i -o main.s`
1. 汇编
		eg: `gcc -c main.s -o main`

gcc [-c|-S|-E] [-std=standard] [-Idir] [-Ldir] [-o outfile] infile... [-lxxx]

    