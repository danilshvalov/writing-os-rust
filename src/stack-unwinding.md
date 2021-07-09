## Стек вызовов
Stacks are last-in, first-out (LIFO) data structures, in other words, the last item pushed (inserted) on the stack is the first item popped (removed) from it. To understand how C++ performs function calls, we should know it's using a stack data structure.

Stack - это структура данных типа "последним вошел, первым вышел" (LIFO), другими словами, последний помещенный (или вставленный) на стек элемент является первый извлекаемым (или удаляемым) элементом. Чаще всего принцип работы стека сравнивают со стопкой тарелок: чтобы взять вторую сверху, нужно снять верхнюю. 

При вызове функции, вызывающая сторона помещает адрес возврата на стек. Когда вызванная функция завершает свою работу, она извлекает адрес возврата со стека и передает управление этому адресу. Если вызванная функция в процессе работы вызывает еще одну функцию, она помещает другой адрес возврата в стек вызов и так далее.

Stack Unwinding
## Раскрутка стека
A call stack is a stack data structure that stores information about the active functions. The call stack is also known as an execution stack, control stack, function stack, or run-time stack. The main reason for having a call stack is to keep track of the point to which each active function should return control when it completes executing. Here, the active functions are those which have been called but have not yet completed execution by returning.

Before we look into the exception aspect of the call stack, let's look at how C++ normally handles function calls and returns. C++ usually handles function calls by placing information on a stack, actually what it is placing is the address of a calling function instruction with its return address. When the called function completes, the program uses that address to decide where to continue the execution. Besides the return address, the function call puts any function arguments on the stack. They are treated as automatic variables. If the called function creates any additional automatic variables, they, too, are added to the stack.

When a function terminates, execution goes to the address stored when the function was called, and the stack for the called function is freed. So, a function normally returns to the function that called it, with each function freeing its automatic variables as it completes. If an automatic variable is a class object, then the destructor for that class is called.

