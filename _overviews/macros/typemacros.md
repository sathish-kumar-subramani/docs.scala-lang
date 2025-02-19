---
layout: multipage-overview
title: Type Macros

discourse: true

overview-name: Macros

languages: [ja]
permalink: /overviews/macros/:title.html
---
<span class="label important" style="float: right;">OBSOLETE</span>

**Eugene Burmako**

Type macros used to be available in previous versions of ["Macro Paradise"](paradise.html),
but are not supported anymore in macro paradise 2.0.
Visit [the paradise 2.0 announcement](https://scalamacros.org/news/2013/08/05/macro-paradise-2.0.0-snapshot.html)
for an explanation and suggested migration strategy.

## Intuition

Just as def macros make the compiler execute custom functions when it sees invocations of certain methods, type macros let one hook into the compiler when certain types are used. The snippet below shows definition and usage of the `H2Db` macro, which generates case classes representing tables in a database along with simple CRUD functionality.

    type H2Db(url: String) = macro impl

    object Db extends H2Db("coffees")

    val brazilian = Db.Coffees.insert("Brazilian", 99, 0)
    Db.Coffees.update(brazilian.copy(price = 10))
    println(Db.Coffees.all)

The full source code of the `H2Db` type macro is provided [at GitHub](https://github.com/xeno-by/typemacros-h2db), and this guide covers its most important aspects. First the macro generates the statically typed database wrapper by connecting to a database at compile-time (tree generation is explained in [the reflection overview]({{ site.baseurl }}/overviews/reflection/overview.html)). Then it uses the <span class="label success">NEW</span> `c.introduceTopLevel` API to insert the generated wrapper into the list of top-level definitions maintained by the compiler. Finally, the macro returns an `Apply` node, which represents a super constructor call to the generated class. <span class="tag">NOTE</span> that type macros are supposed to expand into `c.Tree`, unlike def macros, which expand into `c.Expr[T]`. That's because `Expr`s represent terms, while type macros expand into types.

    type H2Db(url: String) = macro impl

    def impl(c: Context)(url: c.Expr[String]): c.Tree = {
      val name = c.freshName(c.enclosingImpl.name).toTypeName
      val clazz = ClassDef(..., Template(..., generateCode()))
      c.introduceTopLevel(c.enclosingPackage.pid.toString, clazz)
      val classRef = Select(c.enclosingPackage.pid, name)
      Apply(classRef, List(Literal(Constant(c.eval(url)))))
    }

    object Db extends H2Db("coffees")
    // equivalent to: object Db extends Db$1("coffees")

Instead of generating a synthetic class and expanding into a reference to it, a type macro can transform its host instead by returning a `Template` tree. Inside scalac both class and object definitions are internally represented as thin wrappers over `Template` trees, so by expanding into a template, type macro has a possibility to rewrite the entire body of the affected class or object. You can see a full-fledged example of this technique [at GitHub](https://github.com/xeno-by/typemacros-lifter).

    type H2Db(url: String) = macro impl

    def impl(c: Context)(url: c.Expr[String]): c.Tree = {
      val Template(_, _, existingCode) = c.enclosingTemplate
      Template(..., existingCode ++ generateCode())
    }

    object Db extends H2Db("coffees")
    // equivalent to: object Db {
    //   <existing code>
    //   <generated code>
    // }

## Details

Type macros represent a hybrid between def macros and type members. On the one hand, they are defined like methods (e.g. they can have value arguments, type parameters with context bounds, etc). On the other hand, they belong to the namespace of types and, as such, they can only be used where types are expected (see an exhaustive example [at GitHub](https://github.com/scalamacros/kepler/blob/paradise/macros211/test/files/run/macro-typemacros-used-in-funny-places-a/Test_2.scala)), they can only override types or other type macros, etc.

| Feature                        | Def macros | Type macros | Type members |
|--------------------------------|------------|-------------|--------------|
| Are split into defs and impl   | Yes        | Yes         | No           |
| Can have value parameters      | Yes        | Yes         | No           |
| Can have type parameters       | Yes        | Yes         | Yes          |
| ... with variance annotations  | No         | No          | Yes          |
| ... with context bounds        | Yes        | Yes         | No           |
| Can be overloaded              | Yes        | Yes         | No           |
| Can be inherited               | Yes        | Yes         | Yes          |
| Can override and be overridden | Yes        | Yes         | Yes          |

In Scala programs type macros can appear in one of five possible roles: type role, applied type role, parent type role, new role and annotation role. Depending on the role in which a macro is used, which can be inspected with the <span class="label success">NEW</span> `c.macroRole` API, its list of allowed expansions is different.

| Role         | Example                                         | Class | Non-class? | Apply? | Template? |
|--------------|-------------------------------------------------|-------|------------|--------|-----------|
| Type         | `def x: TM(2)(3) = ???`                         | Yes   | Yes        | No     | No        |
| Applied type | `class C[T: TM(2)(3)]`                          | Yes   | Yes        | No     | No        |
| Parent type  | `class C extends TM(2)(3)`<br/>`new TM(2)(3){}` | Yes   | No         | Yes    | Yes       |
| New          | `new TM(2)(3)`                                  | Yes   | No         | Yes    | No        |
| Annotation   | `@TM(2)(3) class C`                             | Yes   | No         | Yes    | No        |

To put it in a nutshell, expansion of a type macro replace the usage of a type macro with a tree it returns. To find out whether an expansion makes sense, mentally replace some usage of a macro with its expansion and check whether the resulting program is correct.

For example, a type macro used as `TM(2)(3)` in `class C extends TM(2)(3)` can expand into `Apply(Ident(TypeName("B")), List(Literal(Constant(2))))`, because that would result in `class C extends B(2)`. However the same expansion wouldn't make sense if `TM(2)(3)` was used as a type in `def x: TM(2)(3) = ???`, because `def x: B(2) = ???` (given that `B` itself is not a type macro; if it is, it will be recursively expanded and the result of the expansion will determine validity of the program).

## Tips and tricks

### Generating classes and objects

With type macros you might increasingly find yourself in a zone where `reify` is not applicable, as explained [at StackOverflow](https://stackoverflow.com/questions/13795490/how-to-use-type-calculated-in-scala-macro-in-a-reify-clause). In that case consider using [quasiquotes]({{ site.baseurl }}/overviews/quasiquotes/intro.html), another experimental feature from macro paradise, as an alternative to manual tree construction.
