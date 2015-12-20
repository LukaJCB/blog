---
layout: post
title:  "Full Stack Scala"
date:   2015-12-19 23:00:25 +0100
categories: scala
---

There's been a lot of talks recently about the "holy grail of web development" of sharing code between the Server and the Client.
I think this is a great idea as writing the same code twice can be expensive as the complexity of your project grows.
I've never been a huge fan of Javascript, but recently a lot of transpilers and extensions to Javascript have emerged, to overcome some of the common pitfalls of Javascript. I've been doing a lot of TypeScript with Angular and Express.js recently and have come to appreciate the working with MEAN, 
I realized, that since I'm already transpiling to Javascript and I'd heard that [Scala.js is no longer experimental][scala.js not exp], that transpiling from Scala might be a good idea.
With [Web Assembly becoming a thing][wasm] transpiling from different languages might just become more and more popular, so I decided to write up a short proof of concept using [Scalatra][scalatra], jQuery and [µPickle][uPickle].

First I thought of a small scenario to model my application after. Imagine you sell Pizzas and want people to order them online.
Now forgetting about security and all that blah, I implemented something small that checks if you order matches the minimum order value.
Take a look.
{% highlight scala %}
package com.ltj.models

case class Topping (name: String, price: Float)

object Topping {
  def apply(name: String): Option[Topping] = name match {
    case "Onion" => Some(Topping(name, 1))
    case "Peppers" => Some(Topping(name, 1.5f))
    case "Ham" => Some(Topping(name, 2.5f))
    case "Pepperoni" => Some(Topping(name, 2))
    case "Olives" => Some(Topping(name, 1))
    case _ => None
  }
}


case class Pizza (toppings: List[Topping]){
	
  def addTopping(topping: Option[Topping]): Pizza = {
    if (topping.isEmpty) this
    else Pizza(toppings :+ topping.get)
  }
}

object Pizza {
  val BasePrice: Float = 3.5f
}


case class Order(pizzas: List[Pizza]) {
  
  val minimumOrderValue = 12.50
  
  def isValidOrder(): Boolean = {
    price() > minimumOrderValue
  }
  
  def price(): Float = {
    val toppingPrices = for {
      pizza <- pizzas
      topping <- pizza.toppings
    } yield topping.price
    toppingPrices.foldLeft(Pizza.BasePrice* pizzas.length)(_ + _)
  }
 
}
{% endhighlight %}

Now for writing the server-side code.
If you're not familiar with Scalatra, it's a small Sinatra-like (obviously) framework, which allows you to quickly create RESTful services (similar to Express).
Scala has built-in xml literals and we can take advantage of that and write pure HTML and map it to a GET-Request. 
Now in a real world application we'd use a templating language like Jade or [Scalate][scalate], but for this app we're just gonna use this shortcut.
{% highlight scala %}
package com.ltj.server

import com.ltj.models.Order
import org.scalatra._
import upickle.default._

class PizzaServlet extends PizzaorderStack {

  get("/") {
    <html>
      <body>
        <h1>Order your Pizza!</h1>
        <form>
          <select id="topping">
            <option value="Margherita">Margherita</option>
            <option value="Onion">Onion</option>
            <option value="Olives">Olives</option>
            <option value="Peppers">Peppers</option>
            <option value="Ham">Ham</option>
            <option value="Pepperoni">Pepperoni</option>
          </select>
          <input type="button" onclick="com.ltj.pizza.Main().submitTopping()" value="Add Topping to Pizza"/><br/>
          <input type="button" onclick="com.ltj.pizza.Main().submitPizza()" value="Add Pizza to Order"/><br/><br/>
          Your current total: <span id="price">0.00</span>$<br/>
          Please select at least 12.50$ before sending your order.
          <input type="button" onclick="com.ltj.pizza.Main().submitOrder()" value="Send Order!"/>
        </form>
        <script src="lib/jquery-1.9.1.min.js"></script>
        <script src="js/scala-2.11/pizza-order-fastopt.js"></script>
      </body>
    </html>
  }

  post("/orders"){
    val order = read[Order](request.body)
    if (!order.isValidOrder()) halt(400)
  }

}

{% endhighlight %}

On a POST-Request to "/orders" we read the incoming JSON, parse it to a Scala Object (using uPickle) and check if it's valid.
If it isn't we'll just halt the request and send a 400 back to the client.
Now as you can probably see, the code above already contains some calls to the client side logic.
If you're unfamiliar with how Scala.js works, check the links below for some great explanations.
Anyway here's the code that will check the order before sending a POST-Request to confirm the order.

{% highlight scala %}

object Main extends JSApp {
	
  var order = Order(List())
  var currentPizza = Pizza(List())
  
  @JSExport
  def submitTopping(): Unit = { 
	  val unvalidatedTopping = jQuery("#topping").`val`.toString()
	  val topping = Topping(unvalidatedTopping)
	  currentPizza = currentPizza.addTopping(topping) 
  }
  
  @JSExport
  def submitPizza(): Unit =  {  
	  order =  Order(order.pizzas :+ currentPizza); 
	  currentPizza = Pizza(List())
	  jQuery("#price").html(order.price().toString())
  }
  
  @JSExport
  def submitOrder(): Unit = { 
	  if (order.isValidOrder()){
	 	  
	 	   val jqxhr = jQuery.post("/orders", write(order),(data: Any) => {
    		 	window.alert("Your order has been processed and will be there shortly!")
    	 })
    	   
	  } else {
	 	  window.alert("Minimum value not reached!")
	  }
  }
  
  def main(): Unit = {
	  
		jQuery.ajaxSetup(js.Dynamic.literal(
			contentType = "application/json"
		))
  }
	

{% endhighlight %}

With the `@JSExport` annotation we can now call the methods using the `onclick` attribute.

So now that we have all this wired up we can start up the application.

![My helpful screenshot]({{ site.url }}/assets/Screenshot.png)

And see there, it works! 
So there we have it. Sharing Scala code between client and the server. 
In my opinion the name "holy grail" does do a lot of justice to this approach. 
However Scala.js probably isn't exactly production ready, although bindings for React and Angular already exist today.
If you're a Scala developer and always wished you could extend your Backend knowledge to the client, you should give it a try. 
The sbt plugins work really well, and I hear integration into Angular and React work quite nicely and combined with the Play! framework it could be very very powerful.
If you're interested, I've prepared a few links.

You cnan find all the code showed here and lots more [in the github repo][git repo].

The basics of Scala.js [http://www.scala-js.org/tutorial/basic/][scalajs basic]

Using Scala.js with jQuery [https://github.com/scala-js/scala-js-jquery][scalajs jquery]

Using sbt to cross compile Scala to the JVM and JS [http://lihaoyi.github.io/hands-on-scala-js/#IntegratingClient-Server][scalajs hands on]

[git repo]: https://github.com/LukaJCB/FullStackScala
[scalajs basic]: http://www.scala-js.org/tutorial/basic/
[scalajs jquery]: https://github.com/scala-js/scala-js-jquery
[scalajs hands on]: http://lihaoyi.github.io/hands-on-scala-js/#IntegratingClient-Server
[scala.js not exp]: http://www.scala-lang.org/news/2015/02/05/scala-js-no-longer-experimental.html
[wasm]: https://brendaneich.com/2015/06/from-asm-js-to-webassembly/
[uPickle]: https://lihaoyi.github.io/upickle-pprint/upickle/
[scalatra]: http://www.scalatra.org/
[scalate]: http://www.scalatra.org/2.4/guides/views/scalate.html



