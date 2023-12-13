# Decoupling Frontend Framwork

In a "flavour-of-the-month" world where new frontend frameworks and libraries are popping up seemingly daily, 
where web developement teams are constantly having to move off of end of life component libraries into the latest version,
how can we build applications in a way that minimizes the impact of migrations from the old to the new?

## Separate Presentation layer from Application and Business layers

The first, and most obvious, separation is in the layers. Vue, React, and other similar libraries exist in the Presentation layer. Angular does also,
but provides other features (mainly dependency injection) that can lead to an integration of your entire application into the Angular framework. This
in my opinion, is a slippery slope.

The Presentation layer, in the context of modern frontend web technologies, boils down mostly to any components, routing, and internationalization in your application. 

Components are responsible for only two things: 
* Output current state to the user
* Accept state modifications from the user

To accomplish this, we need a way for the component to read the current application state and to modify the application state.
