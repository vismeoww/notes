Whenever I have built an executable using C, C++, Rust, I have always wondered. It's just a file! how does the computer know what to do with it? where does all the instructions go? what about things like standard libraries? Things such as `printf` and `scanf` do we include their code as well on every binary that's made? If not how does the computer know how to execute this functions? 

With all of those questions I decided to see how an `elf` format is structured. 
But first for all of that we need some code to compile. Let's start with something small 

```c
#include <stdio.h>
#include <string.h>

const char* true_password = "ThisPasswordLol";

int main(void)
{
    printf("Enter Password: ");
    char password[100];
    scanf("%s", password);
    if(!strcmp(true_password, password)){
      printf("Congrats!!\n");
    } else {
      printf("Wrong password, try again\n");
    }
    return 0;
}

```

let's compile this using 

```bash
gcc main.c -o main
```
