---
layout: post
title:  "Functional Reactive Programming in Angular 2"
date:   2016-04-02 16:00:25 +0100
categories: Angular2
tags: Angular2 FRP RxJS ReactiveX Typescript
---


Today we're going to do a quick run-down on how to build Angular 2 applications using Functional Reactive Programming.
As Angular 2 approaches its release, many developers have already had their hands on the various Alpha and Beta releases.
Angular 2 is not an opinionanted framework and there's dozens of articles, blog posts and tutorials about different kind of architectural styles,
 but one thing I felt was strange, was the lack of articles talking about Functional Reactive Programming in Angular 2.

Unlike Angular 1.x, Angular 2 has unidirectional data flow, which makes reasoning about it that much more easier. 
On top of that Angular 2 is built using RxJS and exposes RxJS `Observables` in the Forms and HTTP API. 

## Introducing Functional Reactive Programming
Functional Reactive Programming (FRP) is a paradigm for creating entire applications with nothing but streams of values over time.
These streams can be anything from mouseclicks and button presses to lists of users and HTTP requests. 
Combining, modifying and subscribing to these event streams is basically all we ever do in this paradigm. 

If you already know Functional Programming, FRP is going to be much easier to grasp. 
The same as in Functional Programming, we want to avoid any kind of mutable state and program by composing pure functions.
Pure functions are functions that do not have any side effects, meaning the function always results in the same return value, when given the same arguments.

All of this will make our code much more concise, we can now program by telling the computer what you want to have, instead of how to get it. 
In that sense FRP is declarative, rather than imperative. 
We can now program at a higher abstraction level, similar to how coding in Angular 1.x featured a much higher abstraction level than coding with jQuery only.
This reduces a lot of common errors and bugs, especially as your applications become more and more complex.

I'm not going to go into any more detail here, but if you'd like to know more, check out this [introduction][frp intro] by `Cycle.js` creator André Staltz.
Basically what FRP is about is this: **Everything is a Stream**, as André phrases it in his intro.

### Everything is an Observable
In RxJS (and all the different implementations of ReactiveX) these streams we talked about are called `Observables`.
In Angular 2, we can get Observables by sending HTTP requests, or using the `Control` API. 
Angular 2 also makes it easy to render these Observables, by subscribing to their value.

Again, I'm not going to go into more depth here, since it would be beyond the scope of this article,
 but if you want, you can check out this [this video][video] by Sergi Mansilla about FRP using RxJS (I can also really recommend his [book!][book]).
 

Without further ado, let's start creating a small sample app, where we can calculate a BMI for a specific person.
The first thing we're going to want to do is creating a model.

{% highlight javascript %}

export class Person {
    name: Observable<string> = Observable.create();
    bmi: Observable<number> = Observable.create();
    category: Observable<string> = Observable.create();
}

{% endhighlight %}

As we can see here, our model is fully comprised of Observables. I wasn't kidding when saying **Everything** is an Observable.

Now it's time to create our template:

{% highlight html %}
<div>
    <h2>{% raw %}{{ person.name | async }}{% endraw %}</h2>
    <form [ngFormModel]="form">
        <label>Name:</label>
        <input type="text" ngControl="name"><br/>
        
        <label>Height (cm):</label>
        <input type="number" ngControl="height"><br/>
        
        <label>Weight (kg):</label>
        <input type="number" ngControl="weight"><br/>
    </form>
    
    <div><strong> Body Mass Index (BMI) = {% raw %}{{ person.bmi | async }}{% endraw %}</strong></div>
    <div><strong> Category: {% raw %}{{ person.category | async }}{% endraw %} </strong></div>
</div>
{% endhighlight %}

Notice the `async` pipe? This is a really neat feature, that allows us to bind Observables to the template. 
So now we have a model, and a template to display and edit the data. The next thing we need is a component to put it all together.

{% highlight javascript %}
@Component({
    selector: "person-bmi",
    templateUrl: 'templates/bmi-unit.html',
    changeDetection: ChangeDetectionStrategy.OnPush
})
export class BmiComponent {
    form: ControlGroup;
    nameControl: Control = new Control("");
    
    @Input('person') person: Person
    
    constructor(fb: FormBuilder) {
        this.form = fb.group({
            "name": this.nameControl,
            "height": new Control(""),
            "weight": new Control("")
        });
    }
    
    ngOnInit(){
        this.person.name = this.nameControl.valueChanges;
        
        this.person.bmi = this.form.valueChanges
        .debounceTime(200)
        .map(value => toBmi(value.weight, value.height))
        .filter(value => value > 0);
        
        this.person.category = this.person.bmi.map(bmi => toCategory(bmi));
    }
}
{% endhighlight %}

Now, I know this is a quite large snippet, but let's walk through it.
The first thing of note, is the change detection strategy. 
By setting it to `OnPush` we get a huge performance boost, because internally Angular doesn't need to do deep equality checks on every browser event anymore.
So we don't just get more organized code, but we also make it faster. Check out [this link][change detection] for more on how Change detection works in Angular 2.

Secondly, notice how we put the initialization inside `ngOnInit` instead of the constructor. 
Since our Person is an marked as an Input field, we don't have access to it inside the constructor, so we resort to `ngOnInit`.

The next thing to notice is that the `valueChanges` field on a `Control` or `ControlGroup` is actually an Observable. 
For our `name` field we don't want to transform the data at all so we just assign it to the plain Observable. 
Angular 2 then subscribes to this Observable when you use the `async` pipe inside your template.

To calculate the bmi-Observable we're doing something more complicated so let's have a closer look:

`this.form.valueChanges.debounceTime(200)`

`debounceTime` emits an element from it's source after any given time (200 ms in our case). This is very useful when we don't want to react to every single key press.
In this case it won't change the BMI value if you just quickly delete a character and add it again directly after. 
It's even more powerful when you use it for something like reacting to key presses with HTTP requests.

`.map(value => toBmi(value.weight, value.height))`

`map` is a very common operator in functional programming, it transforms every single emited value from the stream according to the passed function.
In our case we transform our values for the height and width into a BMI value, by calling the `toBmi` function (omitted here).

`.filter(value => value > 0);`

Lastly we got `filter` which is another very common operator. It tells our Observable to only emit values when they fulfil the given predicate.
We use it here to filter out any values that are `0` or `NaN`.

We then use the bmi to create the `category` Observable which maps a specific bmi to a category like "Normal", "Underweight" or "Obese".

Now when we run this with a `Person` instance we get a fully functioning application that calculates the BMI, written in pure FRP without any external mutable state.
But wait a minute, this app hardly has any state at all. "This was way too easy. I want to see a real CRUD App". 

Alright, alright, let's expand our example to create a list of the component we just created (I know I suck at thinking of example apps, sue me!). 

First, let's create a small template to display our list:

{% highlight html %}

<section>
    <ul>
        <li *ngFor="#person of people | async">
            <person-bmi [person]="person"></person-bmi> 
        </li>
    </ul>
    <hr/>
    <button (click)="addNewPerson()">Add new Person</button>
</section>

{% endhighlight %}

Pretty easy so far, all of the syntax here should be quite easy to understand if you're familiar with Angular 2.
If you're not, check out the [cheat sheet][cheat]. 

Now, it's time to write the component for this template.
{% highlight javascript %}
export class PersonListComponent {
    
    people: Observable<Person[]>;
    
    addNewPerson: () => any;
    
    constructor() {
       
        const peopleSignal = Observable.create(observer => {
            this.addNewPerson = () => observer.next();
        });
        
        this.people =  peopleSignal.map(() => [new Person()])
        .startWith([new Person()])
        .scan((accumulator: Person[], value) => acc.concat(value));
        
    }
    
}
{% endhighlight %}

This is all the code needed for making a list, and see there, we're not mutating any state. Hah! Told ya!
Okay in all seriousness, let's take a look at what we wrote here. 

The `people` field is an Observable again, this time it's an array though. We'll take a look at that in more detail soon.
Then we have the `addNewPerson` function, that get's called whenever we press our button.

Now let's take a look at `peopleSignal`. What we see here is how we create an Observable using `Observable.create()`.
By binding our `addNewPerson` function to call `observer.next()` we ensure that, this Obvservable will emit a value everytime we click the button. 
This is far from pretty, sadly, but it's currently the only way to create one from an event listener in Angular 2 (Let's hope future versions offer something better!).

Then as a last step we transform our `peopleSignal` Observable into one that's actually going to get things done.
The first call to `map` tells our Observable to now emit an Array with exactly one new `Person`. 
The next step is a call to `startWith`. This sets the first value of `people` to be an Array with a single entry. 
Without `startWith` our Observable would be empty when we first start the app.

Now comes the cool part, `scan` is an operator that works very similar to `reduce` or `fold`. 
It processes the whole sequence and returns a single value, for example: `sum` and `average` are very often implemented using `reduce`.
The main difference between `scan` and `reduce` is that `scan` will emit an intermediate value everytime the Observable emits one.
`reduce` on the other hand only emits a value once the sequence ends. 
Our ability to click on our button won't end anytime soon, so we definitely need the `scan` operator for this.
The function we pass into `scan` will make sure, that every new value will be concatenated to our accumulator.


And with that we're done. You can see the full app running [here][live].
We only used the forms API, but the HTTP Api also exposes Observables and can be used in almost the same way.
This app is by no means production ready, but it's enough to give you an idea, of what you can do with Angular and FRP.



### Conclusion

Like `React` before it, Angular 2 doesn't force you to use any specific architecture. 
However, like `React` before it, I feel it's best to take a functional approach as much as we can. 
Using FRP we can coordinate between different components or server backends much more easily, since we don't have to worry as much about managing our state.
In the end, avoiding side effects as much as you can, can really save you time in building a complex reactive program.

Personally, I'm very content with the API the Angular developers have given us. It makes Angular 2 extremely flexible. 
If this post has you intrigued, I suggest you check out [Cycle.js][cycle] and/or [Elm][elm]. 
Both are frameworks that take this kind of approach and expand upon it (although Elm is technically a full language). 
Unlike Angular and React these frameworks specifically want you to use this specific type of architecture. 
It's highly interresting, how far you can take these approaches and I've enjoyed every minute of it. 

If you're already using some form of Reactive Programming with Flux and Redux, you should consider giving RxJS a try. 
I feel it's much easier, and you can use that gained knowledge in any other language that provides the Rx Api (which is basically every common language out there).
You can even find this quote in the [Redux docs][redux]: 

> The question is: do you really need Redux if you already use Rx? Maybe not. It’s not hard to re-implement Redux in Rx. 
> Some say it’s a two-liner using Rx .scan() method. It may very well be!

We've come a long way since Backbone, Knockout and the original AngularJS and the future looks even brighter!
I hope you all enjoyed this piece and consider FRP in your future Angular 2 Project.

You can find all of the code shown here in this [GitHub Repo][git repo].

#### Further reading: 
[Managing state in angular 2][state]

[FRP for Angular2 Developers - RxJs and Observables][frp for angular2]

[Angular 2 Application Architecture - Building Redux-like apps using RxJs][angular2 redux]

[live]: http://lukajcb.github.io/Angular2-FRP/
[angular2 redux]: http://blog.jhades.org/angular-2-application-architecture-building-applications-using-rxjs-and-functional-reactive-programming-vs-redux/
[frp for angular2]: http://blog.jhades.org/functional-reactive-programming-for-angular-2-developers-rxjs-and-observables/ 
[redux]: https://github.com/reactjs/redux/blob/master/docs/introduction/PriorArt.md
[git repo]: https://github.com/LukaJCB/Angular2-FRP
[state]: http://victorsavkin.com/post/137821436516/managing-state-in-angular-2-applications
[frp intro]: https://gist.github.com/staltz/868e7e9bc2a7b8c1f754
[video]: https://www.youtube.com/watch?v=gT6il5fJyAs
[book]: http://www.amazon.com/Reactive-Programming-RxJS-Asynchronous-JavaScript/dp/1680501291/
[cycle]: http://cycle.js.org/
[elm]: http://elm-lang.org/
[change detection]: http://victorsavkin.com/post/110170125256/change-detection-in-angular-2
[cheat]: https://angular.io/docs/ts/latest/cheatsheet.html
