# Complex-Object Change Detection in Angular

 <img width="300" alt="Angular Logo" src="https://d6vdma9166ldh.cloudfront.net/media/images/bd9734c9-def0-47ee-b9ec-027fcfe3cae8.png">

I’m working on an Angular app that has several parent-child components.  Things were going well using the @Input() and @Output() decorators to pass data between components.  Then, my data object increased in complexity when I started using Angular Material tables for data display.  Suddenly, I was getting inconsistent behavior in the view.  Sometimes data were updated; other times partially, or not at all.  

Researching the “change detection loop” and posts on how to force or limit change detection, offered little toward solving my problem.  Blogs on zones, state change triggers, and DOM trees were informational, but none explained the inconsistent behavior.  After hundreds of console.logs and breakpoints, I spotted my problem.  **Object properties that hold an array reference only update when another value property in the object changes.**   I suspect that I read something like that in a post, but to a non computer-science major, it went over my head.

Let’s look at an example. My Angular data model is defined in class Match:

```typescript
  export class Match {
  name: string;
  playerNames: string[];
}
```

My app component creates an instance of Match which has three methods:
1.	addName - To add a name to the playerNames array.
2.	changeMatchName - To change the match name, and 
3.	spread - To be discussed later.

```typescript
export class AppComponent {
  title = "chg-detection";
  match = new Match();
  counter: number;
  constructor() {
    this.match.playerNames = [];
    this.match.name = 'Match 0';
    this.counter = 1;
  }
  addName() {
    this.match.playerNames.push("Bob" + this.match.playerNames.length);
  }
  changeMatchName(counter) {
    this.match.name = "Match " + counter.toString();
    this.counter++;
  }
  spread() {
    this.match.playerNames = [
      ...this.match.playerNames,
      "Chuck" + this.match.playerNames.length
    ];
  }
}
```

The html is:

```html
  <div style="text-align:center">
    <h1>
      Welcome to {{ title }}!
    </h1>
    <img width="300" alt="Angular Logo" src="https://d6vdma9166ldh.cloudfront.net/media/images/bd9734c9-def0-47ee-b9ec-027fcfe3cae8.png">
  </div>
  <div>1: {{match.name}} Player names: {{match.playerNames}} # of players :{{match.playerNames.length}}</div>
  <br>
  <div>2: {{match.name}} Player names: {{match.playerNames}} </div>
  <br>
  <button  color="primary" (click)="addName()">
    Add Player
  </button>
  <button  color="primary" (click)="changeMatchName(counter)">
    Change Match Name
  </button>
  <button  color="primary" (click)="spread()">
    Spread
  </button>
```

Note that **div line 1, contains a value property length that is not included in div line 2**.
  
## The problem
*When the app initiates, it ouputs:*

1: Match 0  Player names:  # of players :0

2: Match 0  Player names:

*Step1: I add a player, line 1 adds the player name but Line 2 does not.*

1: Match 0  Player names: Bob0  # of players :1

2: Match 0  Player names:

*Step 2: I change the match name.  Both lines 1 and 2 update player names to the current state of the Match object*

1: Match 1  Player names: Bob0  # of players :1

2: Match 1  Player names: Bob0

## So what is happening?

Since the addName() method is **pushing** a value on the match.playerNames array, the reference value of the array is not changed.  Only on Line 1 where the “value” of the array length is changed in the view, does Angular’s change detection loop update that <div> of interpolated expressions.  So not only do the component and its view have different change detection loops, but the view is granular in what gets updated. That results in Line 1 with current values of player names, and Line 2 with stale values.

In Step 2, changing the match’s name, which is assigned by value, creates a change detection cycle on both <div> lines resulting in both lines showing the current state of players.

This [StackBlitz](https://stackblitz.com/edit/angular-sa1un1) link provides a working demo.

## Why this inconsistency?

For most apps, I would expect that the current values of an object’s properties are what is anticipated in the view. I have not seen an explanation of why array or other reference objects are not part of change detection, but I suspect it is performance based since navigating the entire tree and especially big arrays can be expensive.  In support of this conclusion, Angular does offer a “ChangeDetectionStrategy.OnPush” strategy to limit change detection to only part of the component tree.  The rationale for .onPush is performance improvement.

## The fix

Several options are possible to force change detection on a reference value.  They all rely on the Angular change detection principle that new objects are always updated.
1.	An ngrx approach with a Redux store.
2.	Use of immutable.js
3.	Use of the ES6 spread operator.
I chose the spread operator as it seemed it is the easiest to implement, to understand, and it is native to javascript.  The spread operator has the form data = {…data, new} where new replaces or adds values to the existing data object and creates a new object value.  More on spread can be found [here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax).

In my example the:
```typescript
this.match.playerNames.push(‘Bob’) 

becomes

this.match.playerNames = [...this.match.playerNames, 'Chuck'];
```
Another approach would be to use a service to retrieve the data object.  In my case, the server data model uses a player id property in match to reference all player attributes.  Including playerNames in that model would add redundant data to the backend datastore or would create a complex angular service using Local Storage.

Read more [here](https://itnext.io/dont-clone-back-end-models-in-angular-f7a749bdc1b0)  



