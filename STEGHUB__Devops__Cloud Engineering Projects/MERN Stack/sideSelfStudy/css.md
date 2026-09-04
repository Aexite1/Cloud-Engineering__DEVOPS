# CSS Essentials: Styling and Page Layout

## Overview

Cascading Style Sheets (CSS) controls how HTML content is presented to users. Instead of placing presentation rules directly into the document structure, CSS provides a separate way to define colours, typography, spacing, dimensions, and layout.

This separation is particularly useful as a project grows because the same style rules can be applied consistently across multiple elements and pages.

## What CSS Controls

CSS can be used to define:

- Text colours and fonts
- Font sizes
- Backgrounds
- Margins and padding
- Borders
- Width and height
- Element display behaviour
- Overall page layout

The result is a clearer separation between **content/structure** and **presentation**.

## Anatomy of a CSS Rule

A CSS rule normally contains a selector followed by one or more declarations.

```css
selector {
    property: value;
}
```

### Understanding the parts

**Selector**  
Identifies the HTML elements that should receive the style. Examples include an element selector such as `h1`, a class selector such as `.my-class`, or an ID selector.

**Property**  
Specifies what aspect of the selected element should change, such as `color`, `margin`, or `font-family`.

**Value**  
Specifies the setting assigned to that property. Depending on the property, this may be a keyword, number, percentage, or another supported CSS value.

For example:

```css
h1 {
    color: blue;
    font-size: 24px;
    background-color: #f0f0f0;
    margin: 10px;
    padding: 20px;
    border: 1px solid black;
    width: 300px;
    height: 150px;
    display: block;
}
```

## Frequently Used Properties

| Property | Purpose |
|---|---|
| `color` | Changes text colour |
| `font-family` | Selects the typeface |
| `font-size` | Controls text size |
| `background-color` | Sets an element's background |
| `margin` | Controls space outside an element |
| `padding` | Controls space inside an element |
| `border` | Defines an element boundary |
| `width` | Sets horizontal size |
| `height` | Sets vertical size |
| `display` | Controls how an element participates in layout |

Property values can be expressed in several forms, including named values, numeric measurements, percentages, and predefined constants.

## Why CSS Is Important

CSS is one of the core technologies of front-end development because it determines how users experience the interface visually. A sound understanding of selectors, declarations, properties, and values provides the foundation for building consistent and maintainable web layouts.
