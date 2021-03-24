# Trying Angular for the first time as a React engineer

Four things I learned the first day I tried Angular.

## Points to cover

1. Component Composition
2. The `ng` cli
   1. Out generate a Component
3. So many files
4. Pipes, date formatting and others
5. Huge list performance (widowing) || HTML syntax.

## Intro

I've been working professionally and using React exclusively for about two and a half years. Recently a applied for my dream job that came with a caveat - they use Angular. No, not AngularJS! That would be a deal breaker for me. Even though the company was cool I did the tech test in React, I thought I'd also try it in Angular. I reached into the docs and time boxed my attempt to 3 hours. Along the way I learned five things that I want to share:

1. The CLI is awesome
2. Even a simple project has a mountain of files
3. Operations called pipes make manipulating data in html really easy
4. Component composition is not very intuitive

I don't think Angular is better or worse than React. It's definitely different and, hopefully I'll get a chance to dig in and see why you'd choose it over React.

## 1. The angular-cli

The CLI is great. It's used for everything from creating a new project with `ng new <project-name>` to adding a new component using `ng generate component <componen-name>`.I've always looked for a similar tool in React. Being able to generate boiler plate code consistently for common tasks is useful for the time strapped dev. I can say for a fact that using the CLI helped me finish my tech test without wasting time figuring out why my components wouldn't run. Turns out you have to declare new compnents in the `app.modules.ts` file for them to be available to other components. It would have taken me forever to figure this out if I'd written the component out myself.

 Above all I love that it's all encapsulated into the same tool. There's no need to remember which binary should run what command. Hopefully it avoids an oversized, unreadable `scripts` object in the `package.json` too. Check out the [docs](https://angular.io/cli/add) for a full list of what the CLI can do.

## 2. Composing Components

Reacts composition model is outstanding. It's intuitive because if you've worked with HTML you can use it. JSX also makes it easy because you're writing what looks like HTML inside your components return statement. In Angular this isn't so simple. Firstly, there is no JSX. All HTML is written as a template so you can't simply import a custom component and wrap it around other components or tags. What confuses me the most is, Angular components have a `selector` to use it in any other component. This is as simple as doing:

```html
<!-- template of any other component -->
<my-custom-component></my-custom-component>
```

But you cannot do this without some effort:

```html
<!-- template of any other component -->
<my-custom-component>
   <my-other-custom-component></my-other-custom-component>
   <div>Nested composed components</div>
</my-custom-component>
```

So much effort in fact, that I cannot find a simple example of how to do it. Hopefully it's easier than it looks.

## 3. Files, Files, Files and more Files

The CLI is amazing and it certainly simplifies the workflow, but! It generates a a huge number of files. Every new component comes with a:

- .spec.ts
- .component.html
- .component.css
- .component.ts

I'm all for breaking very large components into technology specific files - but not in a tiny app. There are other files for environments, polyfills, and no less than 12 config files. No doubt each of them serve a purpose but it can be overwhelming the first time you encounter it.

## 4. Pipes

My favourite feature so far has to be pipes. Pipes are simple functions used in templates to transform data for display. I stumbled across while needing to format a time stamp into a readable date string. It was for a table where the template to render the row looked like this:

```html
<!-- table-container.template.html -->
<tr *ngFor="let sensor of sensorReadings">
  <td>{{ sensor.id }}</td>
  <td>{{ sensor.reading }}</td>
  <td>{{ sensor.reading_ts }}</td>
</tr>
```

 The equivalent React code would be:

 ```javascript
 // TableComponentContainer.js
// .. assume `sensorReadings` is in state
return (
   <Table>
      {sensorReadings.map(sensor => (
         <tr>
           <td>{ sensor.id }</td>
           <td>{ sensor.reading }</td>
           <td>{ sensor.reading_ts }</td>
         </tr>
      ))}
   </Table>
)
```

To format the time stamp I'd need to add `moment` or `dateFns`. Import the relevant function and use it directly in the JSX. This didn't look feasible in the Angular template. How was I going to import a function into it? Pipes solved this in a graceful way - no need for additional libraries either.

```html
<!-- table-container.template.html -->
<tr *ngFor="let sensor of sensorReadings">
  <!-- other cells removed for brevity -->
  
  <td>{{ sensor.reading_ts | date: "MM/d HH:MM"}}</td>
</tr>
```

 It's used by adding a `|` after the data decleration - in this case `sensor.reading_ts`. The name of the pipe is `date`, after which I added the format I needed. You can build your own pipes too. Declare them once in the `app.modules.ts` and use it in any template. Awesome!

## Signing off

I have no doubt that all my initial assumptions will be wrong. There will be plenty to learn if I get the gig, but I'm excited about giving it a go. My belief is that they with teh most experience "wins". And it's more important to use the right technology for the right job rather than pouring fuel on the Javascript Wars.
