This is an unofficial repo that stores precompiled lzz 2.8.2 linux + windows binaries and documentation from the original website converted to markdown. This is useful for distributing projects that require lzz2 to build without having to build lzz2 itself.

For the current source for version 3, see [the official repo](https://github.com/mjspncr/lzz3).

For lzz2 source, see [this mirror](https://github.com/driedfruit/lzz).

# Lzz: The Lazy C++ Programmer's Tool
Lzz is a tool that automates many onerous C++ programming tasks. It can save you a lot of time and make coding more enjoyable. Given a sequence of declarations Lzz will generate your header and source files. For example, given the following code:

```c++
// A.lzz
class A
{
public:
  inline void f (int i) { ... }
  void g (int j = 0) { ... }
};
bool operator == (A const & a1, A const & a2) { ... }
```
Lzz will generate a header file:

```c++
// A.h
#ifndef LZZ_A_h
#define LZZ_A_h
class A
{
public:
  void f (int i);
  void g (int j = 0);
};
inline void A::f (int i) { ... }
bool operator == (A const & a1, A const & a2);
#endif
```

And a source file:

```c++
// A.cpp
#include "A.h"
void A::g (int j) { ... }
bool operator == (A const & a1, A const & a2) { ... }
```

Lzz makes ordinary C++ programming seem low-level. How many times have you neglected to update a header file after editing a source file? This is a silly mistake, yet we do it again and again. C++ forces you to type and maintain duplicate code. Why not let a program generate it for you?

The parser in Lzz is generated by Basil, a backtracking LR(1) parser generator.

Lzz was developed and is actively maintained by Mike Spencer (mike@lazycplusplus.com). 

# Documentation
* [Synopsis](#synopsis)
* [Preprocessor](#preprocessor)
* [Fuzzy Parsing](#fuzzy-parsing)
* [Supported Constructs](#supported-constructs)
* [Lazy Class](#lazy-class)
* [Lazy Functor](#lazy-functor)

## Synopsis
Lzz is a command-line tool. It is run as follows:
```
    lzz options filenames 
```
Options begin with a dash (-); other arguments are processed as filenames. Options can also be specified with the environment variable LZZ_OPTIONS.

The options are:
```
-i
	Output inline function definitions to an inline file. The functions in this file will be inline if the macro LZZ_ENABLE_INLINE is defined when you run your compiler.

-t
	Output function template definitions and class template static member object definitions to a template file. Use this option if you prefer to explicitly instantiate your templates.

-n
	Output inline function template definitions to a template inline file. The functions in this file will be inline if the macro LZZ_ENABLE_INLINE is defined when you run your compiler.

-hx ext
	Set the header file extension. The default is h.

-sx ext
	Set the source file extension. The default is cpp.

-ix ext
	Set the inline file extension. The default is inl.

-tx ext
	Set the template file extension. The default is tpl.

-nx ext
	Set the template inline file extension. The default is tnl.

-hl
	Generate #line directives in the header file.

-sl
	Generate #line directives in the source file.

-il
	Generate #line directives in the inline file.

-tl
	Generate #line directives in the template file.

-nl
	Generate #line directives in the template inline file.

-hd
	Write the header file only if it is different than the previous file.

-sd
	Write the source file only if it is different than the previous file.

-id
	Write the inline file only if it is different than the previous file.

-td
	Write the template file only if it is different than the previous file.

-nd
	Write the template inline file only if it is different than the previous file.

-c
	Create the source file if the header file is created. Use this option if you want to make sure you header file is always self-contained.

-e
	Preprocess all code. Normally code that is captured is not preprocessed, such as function bodies and template arguments. The code is copied verbatim to the appropriate output file. This option will preprocess this code, allowing the end delimiter to appear inside a macro. For example:

		#define BEGIN void func () {
		#define END }

		BEGIN
		// stuff ...
		END

	When preprocessing captured code the preprocessor behaves as a standard preprocessor. #hdr and #end directives, for example, are not allowed, and the #include directive has its standard meaning. The pragma preprocess_block allows you to explicitly set this option on or off within a file.

-o
	Set the output directory for created files. If this directory is different than the directory containing the input file you will likely also want to use the -x option.

-x
	Use absolute filenames in #line directives.

-d
	Delete old files no longer created.

-I directory
	Search directory for #include and #insert files. This option must be repeated to specify several directories. The directories are searched in the same order as the -I options are processed.

-D macro
	Define a function or an object macro. Replacement text must follow the equals sign (=). If the equals sign is not present then the macro is defined to 1. Examples: -DSTR(X)=#X -D NULL_MACRO= -DDEBUG.

-a filename
	Automatically #include a file before the input file is opened. The files are included in the same order as the -a options are processed.

-p
	Preprocess to standard output. Function bodies, expressions and template arguments are not preprocessed, unless the code originates from a macro; this code is represented as a BLOCK token. No files are generated and no files are deleted.

-P
	Preprocess to standard output. All code in the file is preprocessed. No files are generated and no files are deleted.

-E
	Preprocess to standard output. All code in the file, and all code in #include files, is preprocessed. No files are generated and no files are deleted.

-k name
	Set package name. This name is encoded in the header file include guard. This option is necessary if you wish to use the same name for multiple files in your project and you want one file to include the other. Normally in this case you would be using directory names in your include directives.

-da name
	Set the DLL API macro name. The macro must expand to either the export or import Windows DLL declaration specifiers. For example, if MYDLL_API is your macro name you might write (in non-lzz code):

		#ifdef MYDLL_EXPORTS
		#  define MYDLL_API __declspec(dllexport)
		#else
		#  define MYDLL_API __declspec(dllimport)
		#endif

	Here, MYDLL_EXPORTS would be defined when building your DLL.

	This option (and only this option) enables _dll_api semantics. _dll_api is a new keyword: it can be used where you would normally use the DLL API macro in Windows DLL code: in functions, classes and objects that you want exported from your DLL. For example:

		_dll_api void f () {}
		class _dll_api A {}
		_dll_api int i = 0;

	The keyword is also accepted in lazy classes and functors.

	If this option is set then the -dx option must be set too. See below.

	If this option is not set then the _dll_api keyword is ignored, as long as it is used in the proper context.

	Note: define an Lzz macro if you wish to use a different (unique) identifier, for instance: -Ddll_api=_dll_api.

-dx name
	Set the name of the DLL export macro. In the example above this would be MYDLL_EXPORTS. This macro is needed for one special case: explicit class template intantiations. For example, here's how you would export A<int>:

		template class _dll_api A<int>;

	Assuming you use the macro names above, Lzz will generate in the header

		#ifndef MYDLL_EXPORTS
		extern template class MYDLL_API A<int>;
		#endif

	and in the source

		template class MYDLL_API A<int>;

	This is how Windows does it.

-r commands
	Set the syntax error recover commands. See below for the syntax of the commands. The commands can also be specified in the environment variable LZZ_SYNTAX_RECOVER_COMMANDS; this environment variable is processed first.

-v
	Print the values of the above options.

-hr
	Print the help text for the -r option.

-T
	Print the token names usable in the -r option.

-ver
	Print the version number.

-h
-help
	Print the usage and a brief description of the options. 
```
The method the parser uses to recover from syntax errors is configured at runtime. The recover commands are specified with the -r option. The commands must be comma delimited. The recover commands are:
```
    i:token
        Insert a specific token.

    i:[token,token,...]
        Insert a specific sequence of tokens.

    d
        Delete one token.

    d:max_num
        Delete at most max_num tokens.

    d:token
        Delete a specific token.

    r:token
        Replace one token with a specific token.

    r:token:max_num
        Replace at most max_num tokens with a specific token.

    r:token:token
        Replace a specific token (the second) with a specific token (the first).

    r:[token,token,...]
        Replace one token with a specific sequence of tokens.

    r:[token,token,...]:max_num
        Replace at most max_num tokens with a specific sequence of tokens.

    r:[token,token,...]:token
        Replace a specific token with a specific sequence of tokens.

    m
        Move one token after one token to the right (swap token with next token).

    m:max_move
        Move one token after at most max_move tokens to the right.

    m:max_move:max_num
        Move at most max_num tokens after at most max_move tokens to the right. 
```
Where token is a token name. The -T option prints the token names and their lexemes. Token names are not case sensitive.

## Preprocessor

Lzz has a built-in, fully compatible, C++ preprocessor. However, during normal use:

* Function bodies, expressions and template arguments are not preprocessed; this code is copied verbatim to the appropriate output file. (Use the -e option to preprocess all code. See above.)
* Only preprocessor directives in #include files are processed. C++ code is discarded. If you wish to include everything from another file use #insert. 

In earlier versions, a dollar sign ($) was used to denote a directive. Lzz still accepts this use but may not in the future.

Lzz recognizes the following directives:

* `#include`
	* The argument, after macro substitution, must be a filename of the form "filename" or <filename>. Only preprocessor directives in the named file are processed. If you always include the same file consider using the -a option to automatically include the file.
* `#insert`
	* Like #include except (Lzz) code is also processed.
* `#define`
* `#if`
* `#ifdef`
* `#ifndef`
* `#elif`
* `#else`
* `#endif`
* `#line`
* `#error`
	* Same semantics as a standard C++ preprocessor.
* `#warning`
	* The remaining characters on the line are printed to standard output.
* `#pragma`
	* Lzz arguments optionally begin with lzz. If this word is present then Lzz will issue a warning if the pragma is not recognized. The following arguments are recognized:

		* `inl [on | off] [ext]`
			* If on then create an inline file with optional file extension. This pragma should precede any #inl directives.

		* `tpl [on | off] [ext]`
			* If on then create an template file with optional file extension. This pragma should precede any #tpl directives.

		* `tnl [on | off] [ext]`
			* If on then create an template inline file with optional file extension. This pragma should precede any #tnl directives.

		* `once`
			* The enclosing file is not included again.

		* `preprocess_block [on | off]`
			* Preprocess all code. See the -e option above. If an argument is not given on is assumed. 
* `#hdr`
	* The delimited lines are copied verbatim to the header file.
```
#hdr
...
#end
```
* `#src`
	* The delimited lines are copied verbatim to the source file.
```
#src
...
#end
```
* `#inl`
	* The delimited lines are copied verbatim to the inline file, or to the header file if an inline file is not being created.
```
#inl
...
#end
```
* `#tpl`
	* The delimited lines are copied verbatim to the template file, or to the header file if a template file is not being created.
```
#tpl
...
#end
```
* `#tnl`
	* The delimited lines are copied verbatim to the template inline file, or to the header file if a template inline file is not being created. 
```
#tnl
...
#end
```


# Fuzzy Parsing
Unlike a real C++ parser, Lzz does not maintain a type and template name database. Lzz parses using context information only. However, because the C++ grammar is ambiguous this strategy is inadequate in several contexts; fortunately, Lzz can skip over most of them. The following code is captured by the lexer and passed to the parser as a single token.

* Function bodies
* Constructor member initializers
* Array bounds
* Enumerator initializers
	* Restrictions:
		* All , characters must be enclosed in parenthesis. 
		* Example:
			`enum { int I = (Q <int,char>::I), J }`
* Assignment, direct, and brace-enclosed initializers
* Default function arguments
	* Restrictions:
		* All , characters must be enclosed in parenthesis. 
		* Example:
			`void f (int (* g) () = (h <int, int>)) { }`
* Template arguments
	* Restrictions:
		* All < and > characters not delimiting nested template arguments must be enclosed in parenthesis. 
		* Example:
			`A <B <C>, (X < Y)>::D d;`
* Default non-type template arguments
	* Restrictions:
		* All `,`, `<` and `>` characters must be enclosed in parenthesis.
		* Example:
			`template <int (* g) () = (h <int, int>)> class A { }`
			
Outside the above constructs, there are the following restrictions:

* Named parameters must be used in function types.
	* Example:
	* `int f (int (i), int (* g) (int j)) { }`
	* f is a function taking two arguments: an int, and a pointer to a function taking an int and returning an int.
* Named parameters must be used in the catch clauses of function try blocks.
	* Example:
```
	A ()
	try
	  : i (0)
	{ }
	catch (T t)
	{ }
```

* Named parameters must be used in template declarations.
	* Example:
```
	template <class T, int I, template <class R> class S>
	class A
	{ }
```
* The first expression in a direct initializer must be enclosed within a redundant set of parenthesis, or the Lzz keyword _dinit must precede the direct initializer.
	* Example:
```
	A a1 ((X * Y), Z);
	A a2 _dinit (X * Y, Z);
```

## Supported Constructs

Lzz recognizes all names, qualified and unqualified, with and without template arguments, including constructors, destructors, and operator and conversion function names.

Example:

```c++
template <class T>
class A
{
  A () { }
  ~ A <T> () { }
  (A) (A const & a);
  B <int, R::S>::C c;
  typename T::R const & operator + () const { }
  operator class Z * () const { }
  operator Z * () const { }
  friend B::~ B ();
  friend D::operator int * const () const;
  friend B (::operator +) (B & b1, B & b2);
  template <>
  friend typename T::template S <int> T::template h <int> ();
}
```
Lzz recognizes the following C++ constructs:

* namespace definition
	* An unnamed namespace and all enclosed declarations are output to the source file. This rule overrides all others.
  	* The name of a named namespace may be qualified.
		* `namespace A::B { typedef int I; }`
	* is equivalent to:
		* `namespace A { namespace B { typedef int I; } }`
* typedef
	* Typedefs are output to the header file.
* linkage specification
	* A linkage specification of the form
		* `extern string-literal { declaration-seq }`
	* does not affect where the enclosed declarations are output; in this respect it is similar to a named namespace. A linkage specification of the form
		* `extern string-literal declaration`
  	* is treated as an extern specifier for the purpose of determining whether the contained declaration is a definition.
* using directive
* using declaration
	* Using directives and using declarations are output to the source file.
* object declaration
	* Object declarations are output to the header file. Note: an object declaration has either an extern specifier or a linkage specification and does not have an initializer (otherwise it would be a definition).
* object definition
	* An object definition is output to the source file, and, if it is neither static, explicitly const, nor qualified, its declaration is output to the header file.
* enumeration
	* Enumerations are output to the header file.
* function declaration
* function template declaration
	* If a function declaration or a function template declaration is static, it is output to the source file; otherwise, it is output to the header file.
* function definition
	* If a function definition is static, it is output to the source file; otherwise, if the definition is inline:
		* if the option -i is specified, it is output to the inline file, and, if not qualified, its declaration is output to the header file; otherwise, it is output to the header file.
	* If the definition is neither static nor inline, it is output to the source file, and, if not qualified, its declaration is output to the header file.
* function template explicit specialization
	* If function template explicit specialization is static, it is output to the source file; otherwise, if the definition is inline:
		* if the option -i is specified, it is output to the inline file, and its declaration is output to the header file; otherwise, it is output to the header file.
	* If the definition is neither static nor inline, it is output to the source file, and its declaration is output to the header file.
* function template definition
	* If a function template definition is static, it is output to the source file; otherwise, if the definition is inline:
		* if the option -n is specified, it is output to the template inline file, and, if not qualified, its declaration is output to the header file; otherwise, it is output to the header file.
	* If the definition is neither static nor inline:
		* if the option -t is specified, it is output to the template file, and, if not qualified, its declaration is output to the header file; otherwise, it is output to the header file.
* class declaration
* class template declaration
	* Class declarations and class template declarations are output to the header file.
* class definition
* class template explicit specialization
	* Class definitions and class template explicit specializations are output to the header file. Member definitions are output accordingly:
		 * Static objects are defined in the source file.
		 * Function definitions, if inline:
			* if the option -i is specified, they are defined to the inline file; otherwise,
			* they are defined in the header file. 
		* If not inline, they are defined in the source file.
		* Function template definitions, if inline:
			* if the option -n is specified, they are defined to the template inline file; otherwise,
			* they are defined in the header file. 
		* If not inline:
			* if the option -t is specified, they are defined to the template file; otherwise,
			* they are defined in the header file. 
		* Friend function definitions and friend function template definitions are defined inside the class definition. 
	* A declarator cannot follow a class definition. That is, the following constructs are not supported:
```
		typedef struct _A { int i; } A;
		struct B { int i; } b;
```
	* A semi-colon thus is not necessary after the definition.
* class template definition
* class template partial specialization
	* Class template definitions and class template partial specializations are output to the header file. Member definitions are output accordingly:
		* Static objects, if the option -t is specified, are defined in the template file; otherwise, they are defined in the header file.
		* Function definitions and function template definitions, if inline:
			* if the option -n is specified, they are defined to the template inline file; otherwise,
			* they are defined in the header file. 
		* If not inline, they are defined in the header file.
		* Friend function definitions and friend function template definitions are defined inside the class definition. 
* explicit template instantiation
	* Explicit template instantiations are output to the source file. 

## Lazy class

A lazy class is a shorthand notation for a class with a single constructor that only initializes base types and member variables. Here is an example:
```c++
class A (int i, int j = 0) : public Base (j)
{
  int k = 0;
  // ...
}
```
This is equivalent to the following Lzz code:
```c++
class A : public Base
{
  int i;
  int k;
  // ...
public:
  inline explicit A (int i, int j = 0)
	: Base (j), i (i), k (0)
  {
  }
  ~ A ()
  {
  }
}
```
A lazy class is distinguished from an ordinary class by a parameter list following the class name. This is the constructor parameter list. It is also a list of the initial member objects of the class, minus those that are passed to a base type. The parenthesized expression list following a base type are the arguments passed to the base type in the constructor's member initializer list. For example, j above is passed to Base so it does not become a member object of A. The parenthesized expression list after a base type is optional; if it is not present then the base type is not initialized in the constructor's member initializer list.

Member objects declared in a lazy class (like k above) can be initialized with an assignment or direct initializer. The initializer becomes the member initializer for that variable.

Lazy classes are particularly useful when writing visitors. For example:
```c++
namespace
{
  // get class base specifier list
  struct GetBaseSpecList (BaseSpecPtrVector & base_spec_list)
	: NodeVisitor
  {
	// base-spec -> obj-name
	void visit (BaseSpec1Node & node) const
	{
	  // ...
	}
	
	// base-spec -> VIRTUAL access-opt obj-name
	void visit (BaseSpec2Node & node) const
	{
	  // ...
	}
  
	// base-spec -> access virtual-opt obj-name
	void visit (BaseSpec3Node & node) const
	{
	  // ...
	}
  }
}

namespace gram
{
  // get class base specifier list
  void getBaseSpecList (basl::Nonterm & nonterm,
	 BaseSpecPtrVector & base_spec_list)
  {
	nonterm.accept (GetBaseSpecList (base_spec_list));
  }
}
```
A lazy class can be defined anywhere a class can be defined. It can also be defined as a template. For example:
```c++
template <class T = int>
class A (T * t)
{
  // a nested lazy class template
  template <template <class T> class S>
  class B (S <int> const & s)
  {
  }
}
```
## Lazy Functor
A lazy functor is a shorthand notation for a functor. Here is an example:
```c++
void Spud (int i, int j = 0; int k, int l = 0) const
  : public Base (j)
{
}
```
This is equivalent to the following lazy class:
```c++
struct Spud (int i, int j = 0) : public Base (j)
{
  void operator () (int k, int l = 0) const
  {
  }
}
```
A lazy functor is distinguished by the semicolon in the (otherwise) function parameter list. The parameters before the semicolon are the parameters to the constructor, and ones after are the parameters to the operator() function. Any const volatile specifiers after the parameter list apply to the operator() function, not the constructor, and similarly for any throw specification. Function specifiers can be applied to the operator() function, as well as a function try block. For example:
```c++
inline void Spud (int i, int j = 0; int k, int l = 0) const throw (A)
  : public Base (j)
try
{
}
catch (...)
{
}
```
A functor can also be just declared. For example:
```c++
virtual void Spud (int i, int j = 0; int k, int l = 0) const = 0
  : Base (j);
```
If a lazy functor is virtual, the generated class destructor is also virtual.

A lazy functor can be declared or defined anywhere a class can be defined. It can also be declared or defined as a template. For example:
```c++
template <class T>
void Spud (int i, int j = 0; int k, int l = 0) const : public Base (j)
{
}

class A
{
  template <class T>
  virtual void Spud (int i, int j = 0; int k, int l = 0) const = 0
	: public Base (j);
}
```
