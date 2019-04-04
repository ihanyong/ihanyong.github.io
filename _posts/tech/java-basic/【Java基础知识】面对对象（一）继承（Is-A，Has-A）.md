【Java基础知识】面对对象（一）继承（Is-A，Has-A）


Java 里面的继承无处不在。  可以说简单的Java程序也是需要继承的。


Inheritance is everywhere in Java. It's safe to say that it's almost (almost?)
impossible to write even the tiniest Java program without using inheritance. In order
to explore this topic we're going to use the instanceof operator, which we'll discuss
in more detail in Chapter 4. For now, just remember that instanceof returns true
if the reference variable being tested is of the type being compared to. This code:





class Test {
public static void main(String [] args) {
Test t1 = new Test();
Test t2 = new Test();
if (!t1.equals(t2))
System.out.println("they're not equal");
if (t1 instanceof Object)
System.out.println("t1's an Object");
}
}
Produces the output:
they're not equal
t1's an Object


Where did that equals method come from? The reference variable t1 is of type
Test, and there's no equals method in the Test class. Or is there? The second if
test asks whether t1 is an instance of class Object, and because it is (more on that
soon), the if test succeeds.
Hold on…how can t1 be an instance of type Object, we just said it was of type
Test? I'm sure you're way ahead of us here, but it turns out that every class in Java is
a subclass of class Object, (except of course class Object itself). In other words, every
class you'll ever use or ever write will inherit from class Object. You'll always have
an equals method, a clone method, notify, wait, and others, available to use.
Whenever you create a class, you automatically inherit all of class Object's methods.



Why? Let's look at that equals method for instance. Java's creators correctly
assumed that it would be very common for Java programmers to want to compare
instances of their classes to check for equality. If class Object didn't have an equals
method, you'd have to write one yourself; you and every other Java programmer.
That one equals method has been inherited billions of times. (To be fair, equals
has also been overridden billions of times, but we're getting ahead of ourselves.)
For the exam you'll need to know that you can create inheritance relationships
in Java by extending a class. It's also important to understand that the two most
common reasons to use inheritance are
■ To promote code reuse
■ To use polymorphism
Let's start with reuse. A common design approach is to create a fairly generic
version of a class with the intention of creating more specialized subclasses that
inherit from it. For example:
class GameShape {
public void displayShape() {
System.out.println("displaying shape");
}
// more code
}
class PlayerPiece extends GameShape {
public void movePiece() {
System.out.println("moving game piece");
}
// more code
}
public class TestShapes {
public static void main (String[] args) {
PlayerPiece shape = new PlayerPiece();
shape.displayShape();
shape.movePiece();
}
}


Outputs:
displaying shape
moving game piece


Notice that the PlayingPiece class inherits the generic display() method
from the less-specialized class GameShape, and also adds its own method,
movePiece(). Code reuse through inheritance means that methods with generic
functionality (like display())—that could apply to a wide range of different kinds
of shapes in a game—don't have to be reimplemented. That means all specialized
subclasses of GameShape are guaranteed to have the capabilities of the more generic
superclass. You don't want to have to rewrite the display() code in each of your
specialized components of an online game.
But you knew that. You've experienced the pain of duplicate code when you make
a change in one place and have to track down all the other places where that same
(or very similar) code exists.
The second (and related) use of inheritance is to allow your classes to be accessed
polymorphically—a capability provided by interfaces as well, but we'll get to that
in a minute. Let's say that you have a GameLauncher class that wants to loop
through a list of different kinds of GameShape objects, and invoke display() on
each of them. At the time you write this class, you don't know every possible kind
of GameShape subclass that anyone else will ever write. And you sure don't want
to have to redo your code just because somebody decided to build a Dice shape six
months later.
The beautiful thing about polymorphism ("many forms") is that you can treat any
subclass of GameShape as a GameShape. In other words, you can write code in your
GameLauncher class that says, "I don't care what kind of object you are as long as
you inherit from (extend) GameShape. And as far as I'm concerned, if you extend
GameShape then you've definitely got a display() method, so I know I can call it."
Imagine we now have two specialized subclasses that extend the more generic
GameShape class, PlayerPiece and TilePiece:


class GameShape {
public void displayShape()
System.out.println("disp
}
// more code
}
class PlayerPiece extends GameShape {
public void movePiece() {
System.out.println("moving game piece");
}
// more code
}
class TilePiece extends GameShape {
public void getAdjacent() {
System.out.println("getting adjacent tiles");
}
// more code
}

Now imagine a test class has a method with a declared argument type of
GameShape, that means it can take any kind of GameShape. In other words,
any subclass of GameShape can be passed to a method with an argument of type
GameShape. This code
public class TestShapes {
public static void main (String[] args) {
PlayerPiece player = new PlayerPiece();
TilePiece tile = new TilePiece();
doShapes(player);
doShapes(tile);
}
public static void doShapes(GameShape shape) {
shape.displayShape();
}
}
Outputs:
displaying shape
displaying shape
The key point is that the doShapes() method is declared with a GameShape
argument but can be passed any subtype (in this example, a subclass) of GameShape.
The method can then invoke any method of GameShape, without any concern
for the actual runtime class type of the object passed to the method. There are

implications, though. The doShapes() method knows only that the objects are
a type of GameShape, since that's how the parameter is declared. And using a
reference variable declared as type GameShape—regardless of whether the variable
is a method parameter, local variable, or instance variable—means that only the
methods of GameShape can be invoked on it. The methods you can call on a
reference are totally dependent on the declared type of the variable, no matter what
the actual object is, that the reference is referring to. That means you can't use a
GameShape variable to call, say, the getAdjacent() method even if the object
passed in is of type TilePiece. (We'll see this again when we look at interfaces.)
IS-A and HAS-A Relationships
For the exam you need to be able to look at code and determine whether the code
demonstrates an IS-A or HAS-A relationship. The rules are simple, so this should be
one of the few areas where answering the questions correctly is almost a no-brainer.



IS-A
In OO, the concept of IS-A is based on class inheritance or interface
implementation. IS-A is a way of saying, "this thing is a type of that thing." For
example, a Mustang is a type of horse, so in OO terms we can say, "Mustang IS-A
Horse." Subaru IS-A Car. Broccoli IS-A Vegetable (not a very fun one, but it still
counts). You express the IS-A relationship in Java through the keywords extends
(for class inheritance) and implements (for interface implementation).
public class Car {
// Cool Car code goes here
}
public class Subaru extends Car {
// Important Subaru-specific stuff goes here
// Don't forget Subaru inherits accessible Car members which
// can include both methods and variables.
}
A Car is a type of Vehicle, so the inheritance tree might start from the Vehicle
class as follows:
public class Vehicle { ... }
public class Car extends Vehicle { ... }
public class Subaru extends Car { ... }

In OO terms, you can say the following:
Vehicle is the superclass of Car.
Car is the subclass of Vehicle.
Car is the superclass of Subaru.
Subaru is the subclass of Vehicle.
Car inherits from Vehicle.
Subaru inherits from both Vehicle and Car.
Subaru is derived from Car.
Car is derived from Vehicle.
Subaru is derived from Vehicle.
Subaru is a subtype of both Vehicle and Car.
Returning to our IS-A relationship, the following statements are true:
"Car extends Vehicle" means "Car IS-A Vehicle."
"Subaru extends Car" means "Subaru IS-A Car."


And we can also say:
"Subaru IS-A Vehicle" because a class is said to be "a type of" anything further up
in its inheritance tree. If the expression (Foo instanceof Bar) is true, then class
Foo IS-A Bar, even if Foo doesn't directly extend Bar, but instead extends some
other class that is a subclass of Bar. Figure 2-2 illustrates the inheritance tree for
Vehicle, Car, and Subaru. The arrows move from the subclass to the superclass.
In other words, a class' arrow points toward the class from which it extends.



HAS-A

HAS-A relationships are based on usage, rather than inheritance. In other words,
class A HAS-A B if code in class A has a reference to an instance of class B. For
example, you can say the following,
A Horse IS-A Animal. A Horse HAS-A Halter.
The code might look like this:
public class Animal { }
public class Horse extends Animal {
private Halter myHalter;
}
In the preceding code, the Horse class has an instance variable of type Halter, so
you can say that "Horse HAS-A Halter." In other words, Horse has a reference to a
Halter. Horse code can use that Halter reference to invoke methods on the Halter,
and get Halter behavior without having Halter-related code (methods) in the Horse
class itself. Figure 2-3 illustrates the HAS-A relationship between Horse and Halter.


HAS-A relationships allow you to design classes that follow good OO practices
by not having monolithic classes that do a gazillion different things. Classes (and
their resulting objects) should be specialists. As our friend Andrew says, "specialized
classes can actually help reduce bugs." The more specialized the class, the more
likely it is that you can reuse the class in other applications. If you put all the
Halter-related code directly into the Horse class, you'll end up duplicating code
in the Cow class, UnpaidIntern class, and any other class that might need Halter
behavior. By keeping the Halter code in a separate, specialized Halter class, you
have the chance to reuse the Halter class in multiple applications.


FROM THE CLASSROOM


Object-Oriented Design
IS-A and HAS-A relationships and
encapsulation are just the tip of the iceberg
when it comes to object-oriented design.
Many books and graduate theses have been
dedicated to this topic. The reason for the
emphasis on proper design is simple: money.
The cost to deliver a software application has
been estimated to be as much as ten times more
expensive for poorly designed programs. Having
seen the ramifications of poor designs, I can
assure you that this estimate is not far-fetched.
Even the best object-oriented designers
make mistakes. It is difficult to visualize the
relationships between hundreds, or even
thousands, of classes. When mistakes are
discovered during the implementation (code
writing) phase of a project, the amount of code
that has to be rewritten can sometimes cause
programming teams to start over from scratch.
The software industry has evolved to aid the
designer. Visual object modeling languages, like
the Unified Modeling Language (UML), allow
designers to design and easily modify classes
without having to write code first,

because object-oriented components
are represented graphically. This allows
the designer to create a map of the class
relationships and helps them recognize errors
before coding begins. Another innovation
in object-oriented design is design patterns.
Designers noticed that many object-oriented
designs apply consistently from project to
project, and that it was useful to apply the same
designs because it reduced the potential to
introduce new design errors. Object-oriented
designers then started to share these designs
with each other. Now, there are many catalogs
of these design patterns both on the Internet
and in book form.
Although passing the Java certification exam
does not require you to understand objectoriented design this thoroughly, hopefully
this background information will help you
better appreciate why the test writers chose to
include encapsulation, and IS-A, and HAS-A
relationships on the exam.
—Jonathan Meeks, Sun Certified Java Progra



Users of the Horse class (that is, code that calls methods on a Horse instance),
think that the Horse class has Halter behavior. The Horse class might have a
tie(LeadRope rope) method, for example. Users of the Horse class should never
have to know that when they invoke the tie() method, the Horse object turns
around and delegates the call to its Halter class by invoking myHalter.tie(rope).
The scenario just described might look like this:
public class Horse extends Animal {
private Halter myHalter = new Halter();
public void tie(LeadRope rope) {
myHalter.tie(rope); // Delegate tie behavior to the
// Halter object
}
}
public class Halter {
public void tie(LeadRope aRope) {
// Do the actual tie work here
}
}
In OO, we don't want callers to worry about which class or which object
is actually doing the real work. To make that happen, the Horse class hides
implementation details from Horse users. Horse users ask the Horse object to
do things (in this case, tie itself up), and the Horse will either do it or, as in this
example, ask something else to do it. To the caller, though, it always appears that
the Horse object takes care of itself. Users of a Horse should not even need to know
that there is such a thing as a Halter class.