---
layout: post
title: part3.How Does Virtual Destructor Works?
categories: [C++]
description: How Does Virtual Destructor Works
keywords: C++, Virtual Destructor

---

# How Does Virtual Destructor Works?

Finally, we are in the 3rd & last part of this series. We have already discussed how [virtual function](http://www.vishalchovatiya.com/part-1-all-about-virtual-keyword-in-cpp-how-virtual-function-works-internally/) & [virtual class/inheritance](http://www.vishalchovatiya.com/part-2-all-about-virtual-keyword-in-cpp-how-virtual-class-works-internally/) works internally in previous parts. We left one topic i.e. “How Does Virtual Destructor Works?” which we will see now. As usual, before learning anything new I usually start with “Why Do We Need It in the First Place?”

website: http://www.vishalchovatiya.com/part-3-all-about-virtual-keyword-in-c-how-virtual-destructor-works/

## 1.Why Do We Need a Virtual Destructor?

- We will understand this with our earlier example(slightly twisted):

```
class protocol_t {
    private:
        uint8_t *_type;
        // storage ...
    public:
        protocol_t() { _type = new uint8_t; }
        ~protocol_t() { cout<<"~protocol_t"; delete _type; }

        virtual void authenticate(){};
        virtual void connect(){};
        // operations ...
};

class wifi_t : public protocol_t {
    private:
        char *_pass;
        // storage ...
    public:
        wifi_t() { _pass = new char[15]; }
        ~wifi_t() { cout<<"~wifi_t"; delete _pass; }

        virtual void authenticate(){};
        virtual void connect(){};
        // operations ...
};

class bluetooth_t : public protocol_t {
    private:
        char *_pass;
        // storage ...
    public:
        bluetooth_t() { _pass = new char[15]; }
        ~bluetooth_t(){ cout<<"~bluetooth_t"; delete _pass; }

        virtual void authenticate(){};
        virtual void connect(){};
        // operations ...
};

void makeConnection(protocol_t *protocol) {
    protocol->authenticate();
    protocol->connect();
    // Do some tx & rx
    delete protocol;
}    

int main() {
    int prot_type = rand() % 2; 
    makeConnection( (prot_type) ? new wifi_t : new bluetooth_t);
    return 0;
}
```

- So, we have created `wifi_t` & `bluetooth_t` objects dynamically in `main()` and passed it to function `makeConnection()`.
- An objects of `wifi_t`, `bluetooth_t` & `protocol_t` also uses heap memory at construction & destruction time.
- Well, this code compiles & runs fine without any error. But when you run above code, at the time of `delete protocol` line it always calls the destructor of `protocol_t` which you can verify by `~protocol_t` print on console.
- We are freeing only sub-object resources which is `protocol_t` in call of `~protocol_t()` destructor. This means that there is a memory leak as we are not freeing heap memory resource of an object pointed by the pointer `protocol_t` in function `makeConnection()`.
- We even don’t know the type of object `protocol_t` pointer pointed to at the run time.
- Virtual destructors are there to solve this problem. What we have to do is that

```
//...
virtual ~protocol_t() { cout<<"~protocol_t"; delete _type; }
//...
```

- Put keyword **virtual** in front of destructor `~protocol_t()`. Now `delete protocol` line will not directly call the destructor of `protocol`, rather it calls destructor indirectly i.e. using virtual table mechanism(in other words, dynamic dispatch).
- This way it calls the destructor of the object pointed i.e. either `~wifi_t()` or `~bluetooth_t()` by pointer `protocol` & then call the destructor of its base class i.e. `~protocol_t()`.
- Hence, ***virtual destructor uses to delete the object pointed by base class pointer/reference\***.

 ## 2.How Does Virtual Destructor Works?

- The question is how our destructor of a derived class called. The answer is simple it calls the destructor indirectly i.e. using virtual table pointer(`_vptr`). Let’s understand it with the assumption that our pointer `protocol` points to an object of type `wifi_t`.

```
protocol_t *protocol = new wifi_t;
delete protocol;
```

- Here is the memory layout of the object `wifi_t`

```
|                                |          
|--------------------------------| <------ wifi_t class object memory layout
|  protocol_t::_type             |          
|--------------------------------|          
|  protocol_t::_vptr_protocol_t  |----------|
|--------------------------------|          |----------|-------------------------|
|  wifi_t::_pass                 |                     |   type_info wifi_t      |
|--------------------------------|                     |-------------------------|
|                                |                     |   wifi_t::authenticate  |
|                                |                     |-------------------------|
|                                |                     |   wifi_t::connect       |
|                                |                     |-------------------------|
|                                |                     |   wifi_t::~wifi_t       |
|                                |                     |-------------------------|
```

- Now, the statement `delete protocol;` will probably be transformed by a compiler into

```
//...
(*protocol->vptr[3])(protocol);  // Equivalent to `delete protocol;`
//...
```

- Till here it was simple for us to understand how things are working. Because this is similar to a [virtual function](http://www.vishalchovatiya.com/part-1-all-about-virtual-keyword-in-cpp-how-virtual-function-works-internally/) mechanism which we have seen in earlier articles. But the real magic comes when the destructor of a base class i.e. `protocol_t` will be called.
- Which is again done with augmented code by the compiler in derived class destructor & probably be:

```
~wifi_t() { 
    cout<<"~wifi_t"; 
    delete this->_pass;

    // Compiler augmented code ----------------------------------------------------
    // Rewire virtual table
    this->vptr = vtable_protocol_t; // vtable_protocol_t = address of static virtual table

    // Call to base class destructor
    protocol_t::~protocol_t(this); 
    // ----------------------------------------------------------------------------
}
```

- The process of destructing an object takes more operations than those you write inside the body of the destructor. When the compiler generates the code for the destructor, it adds extra code both before and after the user-defined code. Here we have only taken after code for the sake of understanding.
- The same process will happen no matter how long tree up there is.

## 3.Verifying Compiler Augmented Code in Case of the Virtual Destructor

- Although you can not see compiler augmented code(unless if you disassemble it), we can use one hack that will prove that compiler insert the call of base class destructor in a derived class destructor when we use a virtual destructor.
- For the same, consider the following code:

```
class base {
  public:
    virtual ~base()=0;
};

class derived : public base{
  public:
    ~derived(){}
};

int main(){
  base *pbase = new derived;
  delete pbase;
  return 0;
}
```

- When we use ***pure virtual destructor\***, the compiler will throw the following error at the time of linking:

```
exit status 1
/tmp/main-06bc44.o: In function `derived::~derived()':
main.cpp:(.text._ZN7derivedD2Ev[_ZN7derivedD2Ev]+0x11): undefined reference to `base::~base()'
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

- Hence, compiler tried to add the code for base class destructor call in the derived class destructor. But due to unavailability of base class destructor definition linker exited with an error.

## 4.Tricky Example: Guess the Output

- Following example stolen from [stackoverflow](https://stackoverflow.com/questions/7750280/how-does-virtual-destructor-work-in-c):

```
struct base {
   virtual ~base() { f(); }
   virtual void f() { std::cout << "base"; }
};
struct derived : base {
   void f() { std::cout << "derived"; }
};
int main() {
   base * p = new derived;
   delete p;
}
```

- I would recommend you to guess the output first & run it. If you get goosebump, try to interpret below line.
- Standard mandates that the runtime type of the object is that of the class being constructed/destructed at this time, even if the original object that is being constructed/destructed is of a derived type.
- Hence, it prints the output `base`.

## 5.Summary

+ Virtual destructor uses to delete the object pointed by base class pointer/reference
+ Call to virtual destructor is done using dynamic dispatch
+ Compiler augments the derived class destructor code by inserting a call to the base class destructor
+ The runtime type of the object is that of the class being constructed/destructed at this time, even if the original object that is being constructed/destructed is of a derived type
