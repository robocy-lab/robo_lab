---
layout: blog-detail
name: "Reflection in C++. We waited for it! Or did we?"
image: "/assets/blog/reflection.png"
description: "An overview of reflection in C++26 with examples"
tags:
  - "C++"
---
# Reflection in C++. We waited for it! Or did we?
## What is reflection?
Before start talking about reflection in my beloved C++, we need to figure out what is reflection.

> According to [Wikipedia](https://en.wikipedia.org/wiki/Reflective_programming):  
> In computer science, **reflective programming** or **reflection** is the ability of a process to examine, introspect, and modify its own structure and behavior.

In simple words, reflection allows the program to ask itself: "what do I consist of."  
Talking a bit more practical reflection allows us to figure out what fields the object has, which functions and maybe modify itself:
```kotlin
package dev.spgc
  
import kotlin.reflect.KProperty1  
import kotlin.reflect.full.companionObject  
import kotlin.reflect.full.declaredMemberFunctions  
  
class Car(  
    val amountOfWheels: Int,  
    var ownerName: String,  
    var amountOfCrashes: Int,  
    var cost: Double  
){  
  
    fun crash(damage: Double){  
        amountOfCrashes++  
        cost -= damage  
    }  
  
    fun sellTo(newOwner: String){  
        ownerName = newOwner  
    }  
  
    companion object{  
        fun expensiveCar(ownerName: String) = Car(  
            4,  
            ownerName,  
            0,  
            1000000.0  
        )  
    }  
  
}  
  
fun main() {  
    val carFields = Car::class.members.filterIsInstance<KProperty1<Car, *>>()  
    val carMemberFunctions = Car::class.declaredMemberFunctions  
    val carStaticFunctions = Car::class.companionObject?.declaredMemberFunctions  
  
    println("Class car has ${carFields.size} fields and ${carMemberFunctions.size} member functions")  
    println("Fields are")  
    carFields.forEach {  
        println("    ${it.visibility} ${it.name}: ${it.returnType}")  
    }  
  
    println("Member functions are")  
    carMemberFunctions.forEach {  
        val parameter_string = it.parameters.joinToString { param -> "${param.name}: ${param.type}" }  
        println("    ${it.returnType} ${it.name} (${parameter_string})")  
    }  
  
    println("Static functions are")  
    carStaticFunctions?.forEach {  
        val parameter_string = it.parameters.joinToString { param -> "${param.name}: ${param.type}" }  
        println("    ${it.returnType} ${it.name} (${parameter_string})")  
    }  
}
```
Above you can see the simplest example of the reflection in Kotlin.

The example above is extremely simple but shows the idea of how we can use reflection.

### Why do we need it?
The question about a need of reflection is arguable. Some languages like C or C++ (before the 26th standard) survived without the reflection. So what's the purpose?

Assume we have the project with a huge number of domain structures:
```sh
src
|_ domain
   |_ struct1.h
   |_ struct1.c
   |_ struct2.h
   |_ struct2.c
   ...
   |_ struct100500.h
   |_ struct100500.c
...
```
Each of these structures should be serializable to JSON (for some reason we want it). How can we implement this in C?
+ By hand! Each structure should provide a function for JSON serialization. Take into account that all functions in C should have unique names, and voila you have a thousand of boilerplate lines of code that do JSON serialization. Moreover, you can't call them in a loop, because all functions have different names, so you need a global function, which will parse all possible types. One more place for boilerplate!
+ Using XMacro pattern. It's more convenient, but if you have ever worked with a huge macros code base in C, you know that debugging it is a nightmare. In addition, you also should have a function or macro with `__Generic` to be able to unify all calls to serialization

In C++ we have a more bright perspective for this task:
+ Create abstract class: `Serializable`, derive all domain models from it and implement serialization. Still a lot of boilerplates, but at least you don't have a one function which parses all available data types
+ Also use XMacro; for C++ it will be a complete nightmare, since C++ classes are much more complex than C structs. In addition, this approach can't be used for parameterized classes, as templates will be opened only during compilation, not preprocessing – where macros are opened
  But even if you are OK with writing a lot of boilerplate, imagine the following situation: you've decided to move from JSON to YAML. In this case you need to rewrite all the functions you have.

But how can reflection help us with the problem?  
One function to rule them all!  
You implement a function which will take any object, go through all its fields, and store them in JSON/Yaml whatever you want. Deserialization works in the same way: you read JSON and construct the object accordingly. If you've decided to change the format you need to change or add only one function, not 100500.

### So, only parsing JSON?
And no, reflection is powerful tool. Using it we can create a lot of different utills:
+ ORM
+ Config serialization
+ Customizable `to_string` functions
+ Dependencies injectors
+ Test frameworks (move from macros to reflection)
+ Plugin systems
+ RPC and REST automatization

A lot of these utils are already implemented in C++ right now. However, after adding the reflection, a lot of tools can be rewritten with bare C++, without using external generators (like in gRPC), custom preprocessors (like in QT) or macros (like in gTest).

So if you looked for the opportunity to start an open source project in C++ with almost empty field, it is a great time!

## What will be in C++?
The standard is not finished yet. The current post is referencing [P2996R13](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r13.html) proposal, which already has some partial implementations in a few forks of major compilers. So some syntax still can be changed, but I hope the core ideas will remain the same.
### New operators
**Reflection operator**  
Firstly in a new standard, a new operator will be added. The reflection operator will allow you to get the reflection representation of any type:
```c++
constexpr auto r = ^^int; // The reflection representation of int
```
This operator will be used as an entrypoint to a reflection world
****
**Splicer - reflection to code injection operator**  
One of the ways to leave the reflection world is to use `[: :]` operator:
```c++
[: ^^char :] char_var = '*'; // Same as char char_val = '*';
```
This operator takes reflection and uses it in code: using it as a type if reflection was a type, using it like a field of a class if reflection was a field of a class, etc.
****
### New type
A new type will be added: `std::meta::info` the single opaque type which will be returned by `^^` operator and will be used by the majority of the functions in `meta` header
### New library
As mentioned above, a new library with a lot of simple, but still powerful functions will be added. The library is `<meta>`.  I don't think that explaining all the functions will be useful, you can see them in the proposal, so it's better to show a few of them (IMHO the most useful) in examples and describe them there

## Examples
Here there will be a list of examples compiled with [Bloomberg fork of clang](https://github.com/bloomberg/clang-p2996) with commit hash `4c3e6ae840c5edf4ed972a255e9059d1a92d879a`
### Class fields printer
```c++
#include <iostream>  
#include <meta>  
  
  
template <typename T>  
void print_fields()  
{  
    constexpr std::meta::info class_info = ^^T;  
  
    std::cout << std::meta::identifier_of(class_info) << ":\n";  
  
  // This is one of the way to iterate over list of class data members. The proposed `template for` approach doesn't work for some reason with `display_string_of`
    [:  
        std::meta::detail::pretty_printer<char>::expand  
        (  
            std::meta::nonstatic_data_members_of(class_info, std::meta::access_context::unchecked())  
        )  
    :] >> [&]<auto field>()  
        {  
            constexpr std::string_view field_name = std::meta::identifier_of(field);  
  
            constexpr auto field_info = std::meta::display_string_of(std::meta::type_of(field));  
  
            std::string access_modifier;  
  
            if constexpr (std::meta::is_public(field))  
            {  
                access_modifier = "public";  
            }  
            else if constexpr (std::meta::is_protected(field))  
            {  
                access_modifier = "protected";  
            }  
            else  
            {  
                access_modifier = "private";  
            }  
  
            std::cout << "  " << access_modifier << " " << field_name << ": " << field_info << " " << "\n";  
        };  
}

struct Car  
{  
    int cost;  
    int amountOfWheels;  
    
    Car()  
	: cost{0}  
	, amountOfWheels{}  
	, owner{} 
	{}
  
private:  
    std::string owner;  
};  
  
using Cars = struct_of_vec_t<Car>;  
  
int main(int argc, char *argv[])  
{  
    print_fields<Car>();
}
```
**STDOUT**:
```
Car:
  public cost: int 
  public amountOfWheels: int 
  private owner: basic_string<char, char_traits<char>, allocator<char>> 
```
This is a classical example of reflection usage. Such functions can be extremely helpful in debugging and in serialization.

Here you can see a demo of a few functions and approaches:
1.  `std::meta::identifier_of` allows to get the `std::string_view` with name of the reflected object (but for some reason it doesn't work for reflections of primitive types)
2. `std::meta::display_string_of` allows to get the `std::string_view` with name of the reflected object, in a proposal it's said that this function should output human-readable and pretty output, however for now the only difference between this function and `std::meta::identifier_of` that this function works with reflections of primitive types.
3. `std::meta::nonstatic_data_members_of` allows to get `std::vector<std::meta::info>` of non-static fields of class basing on the context (which will be described a bit bellow)
4. `std::meta::is_public/protected/private()` allows to know the access qualifier of the field
5. `std::meta::type_of` allows to get type reflection of the reflected value
   
As you may notice to iterate over the members of the class I used pretty strange functionality:
```c++
    [:  
        std::meta::detail::pretty_printer<char>::expand  
        (  
            std::meta::nonstatic_data_members_of(class_info, std::meta::access_context::unchecked())  
        )  
    :] >> [&]<auto field>()  
	{ 
	// Some code 
	}
```
This happens because on the one hand the  `std::meta::nonstatic_data_members_of` returns `std::vector<std::meta::info>`, which cannot be used in const-eval context which is required by `std::meta::info`, but on the other hand the proposed `template for` construction for some reason doesn't work correctly with `display_string_of` function (leads to crash).  
**IMPORTANT**: most probably it won't be part of the standard, but rather a temporary hack in the Bloomberg for of clang

Speaking about contexts (as I promised above), the proposal provides the idea of contexts to access members of classes with different qualifiers.  There are three contexts:
1. `current` which inherits the current access rights (e.g. you have access rights like you call the class fields directly in the code)
2. `unprivileged` - can see only public fields
3. `unchecked` - can see everything

### Unified debug function
```c++
#include <iostream>
#include <meta>  
  
  
template <typename T>  
void print_object(T t)  
{  
    constexpr std::meta::info class_info = ^^T;  
  
    std::cout << std::meta::identifier_of(class_info) << ":\n";  
  
  
    template for (constexpr auto field : std::define_static_array(std::meta::nonstatic_data_members_of(class_info, std::meta::access_context::unchecked())))  
    {  
        constexpr std::string_view field_name = std::meta::identifier_of(field);  
        std::cout << "  " << field_name << " = "  << t.[: field :] << "\n";  
    }  
}

struct Car  
{  
    int cost;  
    int amountOfWheels;  
  
    Car(std::string owner, int cost, int amountOfWheels)  
    : cost(cost)  
    , amountOfWheels(amountOfWheels)  
    , owner(owner) {}  
  
private:  
    std::string owner;  
};  
  
int main(int argc, char *argv[])  
{  
    Car car{50, 10, "Bob"};  
    print_object(car);
}
```
**STDOUT**:
```
Car:
  cost = 50
  amountOfWheels = 10
  owner = Bob
```
This example is a continuation of the previous. And also can be useful in debugging
1. As you can see the function can even access the private fields of the class, that's all thanks to `Splicer` operator.
2. Also, there is correct iterating over the members of the class using `template for` which generates const-eval context inside (simply speaking (which is not truly correct) the `template for` statement will copy-paste its body changing all depending on data types accordingly, this is a powerful thing not only in terms of reflection, but also in terms of unpacking `std::tuple`, for example)
3. The `std::define_static_array` promotes a compile-time array to static storage, which allow us to use containers like `std::vector` in the const-eval context (which is crucial for expansion statement like `template for`)
### Dictionary (map) serializer
```c++
#include <any>  
#include <map>  
#include <meta>

template <typename T>  
T dict_serializer(std::map<std::string, std::any> dict)  
{  
    using namespace std::meta;  
  
    T result{};  
  
    constexpr auto ctx = access_context::unchecked();  
  
    template for (constexpr auto member : std::define_static_array(nonstatic_data_members_of(^^T, ctx)))  
        {  
            std::string_view name = identifier_of(member);  
            if (name.empty())  
            {  
                continue;  
            }  
  
            auto it = dict.find(std::string{name});  
            if (it == dict.end())  
            {  
                continue;  
            }  
  
            using MemberType = typename[: type_of(member) :];  
  
            if (auto p = std::any_cast<MemberType>(&it->second))  
            {  
                result.[: member :] = *p;  
            }  
            else  
            {  
                throw std::runtime_error("dict_serializer: wrong type for field '" + std::string{name} + "'");  
            }  
        };  
  
    return result;  
}

struct Car  
{  
    int cost;  
    int amountOfWheels;  
  
    Car()  
    : cost{0}  
    , amountOfWheels{}  
    , owner{} {}  
  
    Car(std::string owner, int cost, int amountOfWheels)  
    : cost(cost)  
    , amountOfWheels(amountOfWheels)  
    , owner(owner) {}  
  
private:  
    std::string owner;  
};  
  
using Cars = struct_of_vec_t<Car>;  
  
int main(int argc, char *argv[])  
{  
    print_fields<Car>();  
  
    std::map<std::string, std::any> dict = {  
        std::pair("cost", std::any(50)),  
        std::pair("amountOfWheels", std::any(10)),  
        std::pair("owner", std::string("Bob")),  
    };  
  
    Car car = dict_serializer<Car>(dict);  
  
    print_object(car);
}
```

**STDOUT**
```
Car:
  cost = 50
  amountOfWheels = 10
  owner = Bob
```
Nothing new here, but it's a great example, how unified JSON serialization/deserialization can be implemented.

### New types generation
```c++
#include <meta>  
#include <vector>  
  
  
template <typename T>  
struct struct_of_vec  
{  
    struct impl;  
  
    consteval  
    {  
        auto ctx = std::meta::access_context::unchecked();  
  
        std::vector<std::meta::info> old_members = std::meta::nonstatic_data_members_of(^^T, ctx);  
        std::vector<std::meta::info> new_members = {};  
        for (std::meta::info member : old_members) {  
            auto vec_type = std::meta::substitute(^^std::vector, {  
                type_of(member),  
            });  
            auto mem_descr = std::meta::data_member_spec(vec_type, {.name = identifier_of(member)});  
            new_members.push_back(mem_descr);  
        }  
  
        std::meta::define_aggregate(^^impl, new_members);  
    };  
  
};  
  
template <typename T>  
using struct_of_vec_t = struct_of_vec<T>::impl;

struct Car  
{  
    int cost;  
    int amountOfWheels;  
  
    Car()  
    : cost{0}  
    , amountOfWheels{}  
    , owner{} {}  
    
private:  
    std::string owner;  
};  
  
using Cars = struct_of_vec_t<Car>;  
  
int main(int argc, char *argv[])  
{   
    print_fields<Cars>();
}
```
**STDOUT**
```
impl:
  public cost: vector<int, allocator<int>> 
  public amountOfWheels: vector<int, allocator<int>> 
  public owner: vector<basic_string<char, char_traits<char>, allocator<char>>, allocator<basic_string<char, char_traits<char>, allocator<char>>>> 
```
Here is an example of how reflection can be used for a generation of new types based on existing ones. Such approaches can help a lot in the fields, where third party code generators are used right now (like gRPC).
1. `std::meta::substitute` allows to substitute the template value of the templated object (class/function). Of course all of these in terms of reflection
2. `std::meta::data_member_spec` allows to generate data member description, which can be used by `std::meta::define_aggregate`
3. `std::meta::define_aggregate` allows to populate incomplete class/struct/union with data members using their reflections (which can be constructed with `std::meta::data_member_spec`)
   
Also, there is a one more important concept of C++26 - `consteval` block which block of code which will be executed in compile-time (something similar to `static_assert((/*Some code*/,true))`), that's why we are allowed to interact with `std::meta::info` in a simple `for` loop.
## Consideration about the static nature of reflection
Unlike reflection in most of the languages with side-runtime (like Python) the reflection in C++ is static and should be available in compile time.  
It brings some limitations.  
There are small inconveniences like: string representations are only `std::string_view` not `std::string` or iterating through fields can be tricky sometimes. But all of these are just unusual syntax that should be usual for C++ programmer 😉.  
However, unusual syntax is the least important drawback of static reflection. With this approach you can't make code generation based on config without recompilation and side scripts.  
In addition, all the data required for the reflection should be available in the compile time. This means that the majority of the utils mentioned at the beginning of the post and examples should be created in a ~~(single)~~ header library manner.

## Conclusion
So what's next? Will the reflection improve our life? On the one hand, we haven't received a silver bullet which can solve all the problems. But it looks like there is no such one!
The proposed reflection, if it manages to get to the final version of the standard and to the compilers, definitely will make our lives easy. No more code generation scripts and tools everywhere, small and beautiful parsers and less usage of macros.
In addition to this, C++ even with reflection will remain an extremely fast language all thanks to the static nature of the proposed reflection approach.
****
The main question right now is when to expect reflection in the compilers?
1. First, as mentioned above, "Bloomberg" has their fork of clang, which you can already use to play with the reflection
2. The GCC already [has](https://gcc.gnu.org/projects/cxx-status.html) implementation of most of the proposals of the 26th standard, however reflection and `define_static_{string,object,array}` is not implemented yet, but looks like it will take little time!

So it's a great time to touch the reflection and maybe start to create libraries even with the current unstable API, like [simdjson](https://github.com/simdjson/simdjson/blob/master/include/simdjson/compile_time_json.h) team.

## Links
1. The proposal - https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2996r13.html
2. Bloomberg compiler - https://github.com/bloomberg/clang-p2996
3. GodBolt (compiler explorer):
    1. clang - https://godbolt.org/z/71647q5Mo
    2. edg - https://godbolt.org/z/4hK564scs
4. Template repository to play with reflection - https://github.com/SPGC/clang-p2996-environment