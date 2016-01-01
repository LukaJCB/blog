---
layout: post
title:  "Full Stack Scala Part 2: Persistence with MongoDB"
date:   2015-12-31 14:00:25 +0100
categories: Scala
tags: Scala Scala.js Scalatra Mongodb Casbah REST
---

In the [last post] we talked about setting up a tiny application that shares Scala code across the Server and Client. 
However no real application is complete without persistence or a database. I'd like to share some quick code snippets to enhance the small proof-of-concept from last time.
As a database I've chosen [MongoDB][mongo], because it fits our needs well and it's really simple to set up!
We could also use any other database be it relational or another style of NoSQL DB, not trying to start any discussions on BASE vs ACID or the like.
If you're not familiar with MongoDB, it's a document-oriented database, that saves JSON-like (BSON - Binary JSON) documents.
You can try it out real quick in the Mongo shell, which is also a full-on Javascript interpreter. If you want to go into detail I can also recommend [this book][book].
We'll start out by adding [Casbah][casbah] to our sbt dependencies. Casbah is a small toolkit for working with Mongo in Scala.
Now we can start using Mongo by connecting inside our ScalatraBootstrap.

{% highlight scala %}
import com.ltj.server._
import org.scalatra._
import javax.servlet.ServletContext
import com.mongodb.casbah.Imports._

class ScalatraBootstrap extends LifeCycle {
  override def init(context: ServletContext) {
    val mongoClient =  MongoClient()
    val mongo = mongoClient("pizza")
    context.mount(new PizzaServlet(mongo), "/*")
  }
}
{% endhighlight %}

With casbah we can create a client with default settings (port 27017 and localhost). 
We can then connent to a database by calling the client. In our case we'll connect to the database "pizza".
Then we insert the database into the constructor of our servlet. 
In our servlet we'll use this instance to persist our orders, query them and delete them.
These 3 cases map very well to the HTTP methods and we'll keep using only JSON to transfer our data.
{% highlight scala %}

class PizzaServlet(db: MongoDB) extends PizzaorderStack {

  post("/orders"){
    val json = request.body
    val order = read[Order](json)
    if (!order.isValidOrder()) halt(400)

    val orders = db("orders")
    orders += JSON.parse(json).asInstanceOf[DBObject]

  }

  get("/orders"){
    val orders = db("orders").find()
    JSON.serialize(orders.toList)
  }

  delete("/orders/:id"){
    val id = new ObjectId(params("id"))
    val orders = db("orders")
    orders.remove(MongoDBObject("_id" ->  id))
  }

}

{% endhighlight %}

As we can see, working with Mongo and JSON is pretty simple. We're just pushing our JSON data straight to the database without normalization.
Now I did a small test with our current client to see if the POST-Request to orders saves the order and voila! It works!

![clientTest]({{ site.url }}/assets/mongoclient.png)


The "find"-method returns a cursor which we can easily convert to a List or another Scala collection and then just serialize it to JSON.
Deleting is a little bit more tricky. Mongo has it's own mechanic of keeping a unique id for each entry of the collection called the "ObjectId". 
If you've expiremented with the Mongo shell or worked with Mongo before you'll familiar with it, but basically it's a 12-byte BSON Object which Mongo will add to your document if you don't specify your own unique identifier.
It's pretty clever and you can check out some of the details [here][object-id]. 
So once we've converted our id-string to an ObjectId we can then easily remove the specified document.
Now we can start expanding our client. We'll start by making another HTML template where we want to list all of our orders.

{% highlight html %}
<html lang="en">
<head>
	<meta charset="UTF-8">
	<title>Orders</title>
</head>
<body onload="com.ltj.pizza.Main().showOrders()">
	<h1>All Orders:</h1>
	<ul id="orders">
	
	</ul>
	<script src="lib/jquery-1.9.1.min.js"></script>
	<script src="js/scala-2.11/pizza-order-fastopt.js"></script>
</body>
</html>

{% endhighlight %}


Pretty simple so far, now let's create some code, that retrieves all the orders via GET-Request and then appends it into our "orders" list.




{% highlight scala %}
@JSExport
def showOrders(): Unit = {
  jQuery.get(url = "/orders",  success = (data: String) => {
    val orders = read[Seq[Order]](data)
    orders.foreach(order => {
	  val list = jQuery("<ul>")
	  order.pizzas.foreach { pizza => list.append(jQuery("<li>").text(pizza.toString())) }
	  jQuery("#orders").append(jQuery("<li>").append(list))
    })
  })
}
  

{% endhighlight %}

We just parse the json back to a Seq of Orders and use it to create new `ul` and `li` elements.
This is also real simple and probably not the best example for creating a good overview for our orders, but we also don't want to do a lot of DOM-Manipulation.
Maybe in another part of this series, we could recreate this with more functionality using Angular or React.
However let's take a look at the result!

![It works!]({{ site.url }}/assets/itworks.png)

Yay! A litte bit shabby but now we have a very easy-to-work-with setup from which we could expand on. 
We've already talked about what our next step could look like, so I'd like to end this post with a little video I found on the topic of Scala.js
[Have fun!][video]

You can find all the code showed here and lots more [in the github repo][git repo].

[object-id]: https://docs.mongodb.org/manual/reference/object-id/
[git repo]: https://github.com/LukaJCB/FullStackScala
[video]: https://www.youtube.com/watch?v=n1GgVWOThhY
[last post]: http://lukajcb.github.io/blog/scala/2015/12/19/full-stack-scala.html
[mongo]: https://www.mongodb.org/
[book]: http://shop.oreilly.com/product/0636920028031.do
[casbah]: https://mongodb.github.io/casbah/