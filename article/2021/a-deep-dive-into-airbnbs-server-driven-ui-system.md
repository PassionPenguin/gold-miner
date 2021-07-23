> * 原文地址：[A Deep Dive into Airbnb’s Server-Driven UI System](https://medium.com/airbnb-engineering/a-deep-dive-into-airbnbs-server-driven-ui-system-842244c5f5)
> * 原文作者：[Ryan Brooks](https://medium.com/@rbro112)
> * 译文出自：[掘金翻译计划](https://github.com/xitu/gold-miner)
> * 本文永久链接：[https://github.com/xitu/gold-miner/blob/master/article/2021/a-deep-dive-into-airbnbs-server-driven-ui-system.md](https://github.com/xitu/gold-miner/blob/master/article/2021/a-deep-dive-into-airbnbs-server-driven-ui-system.md)
> * 译者：
> * 校对者：

# A Deep Dive into Airbnb’s Server-Driven UI System

How Airbnb ships features faster across web, iOS, and Android using a server-driven UI system named Ghost Platform 👻.

By [Ryan Brooks](https://www.linkedin.com/in/rbro112/)

![](https://miro.medium.com/max/1400/0*CedYKpSYMIGEiX7m)

### Background: Server-Driven UI

Before we dive into Airbnb’s implementation of server-driven UI (SDUI), it’s important to understand the general idea of SDUI and how it provides an advantage over traditional client-driven UI.

In a traditional world, data is driven by the backend and the UI is driven by each client (web, iOS, and Android). As an example, let’s take Airbnb’s listing page. To show our users a listing, we might request listing data from the backend. Upon receiving this listing data, the client transforms that data into UI.

This comes with a few issues. First, there’s listing-specific logic built on each client to transform and render the listing data. This logic becomes complicated quickly and is inflexible if we make changes to how listings are displayed down the road.

Second, each client has to maintain parity with each other. As mentioned, the logic for this screen gets complicated quickly and each client has their own intricacies and specific implementations for handling state, displaying UI, etc. It’s easy for clients to quickly diverge from one another.

Finally, mobile has a versioning problem. Each time we need to add new features to our listing page, we need to release a new version of our mobile apps for users to get the latest experience. Until users update, we have few ways to determine if users are using or responding well to these new features.

### The Case for SDUI

What if clients didn’t need to know they were even displaying a listing? What if we could pass the UI directly to the client and skip the idea of listing data entirely? That’s essentially what SDUI does — we pass both the UI and the data together, and the client displays it agnostic of the data it contains.

Airbnb’s specific SDUI implementation enables our backend to control the data and how that data is displayed across all clients at the same time. Everything from the screen’s layout, how sections are arranged in that layout, the data displayed in each section, and even the actions taken when users interact with sections is controlled by a single backend response across our web, iOS, and Android apps.

## SDUI at Airbnb: The Ghost Platform 👻

The Ghost Platform (GP) is a unified, opinionated, server-driven UI system that enables us to iterate rapidly and launch features safely across web, iOS, and Android. It is called Ghost because our primary focus is around ‘**G**uest’ and ‘**Host**’ features, the two sides to our Airbnb apps.

GP provides web, iOS, and Android frameworks in each client’s native languages (Typescript, Swift, and Kotlin, respectively) that enable developers to create server-driven features with minimal setup.

The core feature of GP is that features can share a library of generic sections, layouts, and actions, many backward compatible, enabling teams to ship faster and move complicated business logic to a central location on the backend.

### A Standardized Schema

The backbone of the Ghost Platform is a standardized data model that clients can use to render UI. To make this possible, GP leverages a shared data layer across backend services using a unified data-service mesh, called [Viaduct](/airbnb-engineering/taming-service-oriented-architecture-using-a-data-oriented-service-mesh-da771a841344).

The key decision that helped us make our server-driven UI system scalable was to use a single, shared GraphQL schema for Web, iOS, and Android apps — i.e., we’re using the same schema for handling responses and generating strongly typed data models across all of our platforms.

We’ve taken time to generalize the shared aspects of different features and to account for each page’s idiosyncrasies in a consistent, thoughtful way. The result is a universal schema that’s capable of rendering all features on Airbnb. This schema is powerful enough to account for reusable sections, dynamic layouts, subpages, actions, and more, and the corresponding GP frameworks in our client applications leverage this universal schema to standardize UI rendering.

## The GP Response

The first fundamental aspect of GP is the structure of the overall response. There are two main concepts used to describe UI in a GP response: sections and screens.

![](https://miro.medium.com/max/1400/0*Qx23_meafmhc7-xQ)
<small>Figure 1. *How users see Airbnb features on GP vs. how GP sees those same features as screens and sections.*</small>

* **Sections:** Sections are the most primitive building block of GP. A section describes the data of a cohesive group of UI components, containing the exact data to be displayed — already translated, localized, and formatted. Each client takes the section data and transforms it directly into UI.
* **Screens:** Any GP response can have an arbitrary number of screens. Each screen describes the layout of the screen and, in turn, where sections from the `sections` array will appear (called placements). It also defines other metadata, such as how to render sections — e.g., as a popover, modal, or full-screen — and logging data.

```graphql
interface GPResponse {
  sections: [SectionContainer]
  screens: [ScreenContainer]
  
  # ... Other metadata, logging data or feature-specific logic
}
```
<small>Figure 2. A sample of the GP Response GraphQL schema.</small>

A feature’s backend built with GP will implement this `GPResponse` (fig. 2) and populate the screens and sections depending on their use case. GP client frameworks on web, iOS, and Android provide developers standard handling to fetch a `GPResponse` implementation and translate that to UI with minimal work on their part.

## Sections

Sections are the most basic building block of GP. The key feature of GP sections is that they are entirely independent of other sections and the screen on which they are displayed.

By decoupling sections from the context around them, we gain the ability to reuse and repurpose sections without worrying about a tight coupling of business logic to any specific feature.

### Section Schema

In GraphQL schema, GP sections are a union of all possible section types. Each section type specifies the fields they provide to be rendered. Sections are received in a `GPResponse` implementation with some metadata and are provided through a `SectionContainer` wrapper, which contains details about the section’s status, logging data, and the actual section data model.

```graphql
# Example sections
type HeroSection {
  # Image urls
  images: [String]!
}

type TitleSection {
  title: String!,
  titleStyle: TextStyle!
  
  # Optional subtitle
  subtitle: String
  subtitleStyle: TextStyle
  
  # Action to be taken when tapping the optional subtitle
  onSubtitleClickAction: IAction
}

enum SectionComponentType {
  HERO,
  TITLE,
  PLUS_TITLE,
  
  # ... There's alot of these :)
}

union Section = HeroSection
  | TitleSection
  | # ... More section data models
  
# The wrapper that wraps each section. Responsible for metadata, logging data and SectionComponentType
type SectionContainer {
  id: String!
  
  # The key that determines how to render the section data model
  sectionComponentType: SectionComponentType
  
  # The data for this specific section
  section: Section
  
  # ... Metadata, logging data & more
}
```
<small>*Figure 3. A snippet of what our section GraphQL schema*</small>

One important concept to touch on is `SectionComponentType`. `SectionComponentType` controls *how* a section’s data model is rendered. This enables one data model to be rendered in many different ways if needed.

For example, the two `SectionComponentType`s `TITLE` and `PLUS_TITLE` might use the same backing `TitleSection` data model, but the `PLUS_TITLE` implementation will use Airbnb’s Plus-specific logo and title style to render the `TitleSection`. This provides flexibility to features using GP, while still promoting schema and data reusability.

![](https://miro.medium.com/max/1400/0*i-Mi5mngBjWXQkFk)
<small>Figure 4. *Example of rendering a TitleSection data model differently using SectionComponentType.*</small>

### Section Components

Section data is transformed into UI through “Section Components”. Each section component is responsible for transforming a data model and a `SectionComponentType` into UI components. Abstract section components are provided by GP on each platform in their native languages (i.e., Typescript, Swift, Kotlin) and can be extended by developers to create new sections.

Section components map a section data model to **one** unique rendering and therefore pertain only to one `SectionComponentType`. As mentioned previously, sections are rendered without any context from the screen they are on or the sections around them, so each section component has no feature-specific business logic provided to it.

I’m an Android developer, so let’s take an Android example (and because Kotlin is great 😄). To build a title section, we have the code snippet seen below (fig. 5). Web and iOS have similar implementations — in Typescript and Swift respectively — for building section components.

```kotlin
// This annotation builds a Map<SectionComponentType, SectionComponent> that GP uses to render sections
@SectionComponentType(SectionComponentType.TITLE)
class TitleSectionComponent : SectionComponent<TitleSection>() {
 
    // Developers override this method and build UI from TitleSection corresponding to TITLE
    override fun buildSectionUI(section: TitleSection) {

        // Text() Turns our title into a styled TextView
        Text(
            text = section.title,
            style = section.titleStyle
        )

        // Optionally build a subtitle if present in the TitleSection data model
        if (!section.subtitle.isNullOrEmpty() {

            Text(
                text = section.subtitle,
                style = section.subtitleStyle
            )
        }
    }
}
```
<small>Figure 5. A Kotlin example of a section component.</small>

GP provides many “core” section components, such as our example `TitleSectionComponent` above (fig. 5), meant to be configurable, styleable, and backward compatible from the backend so we can adapt to any feature’s use case. However, developers building new features on GP can add new section components as needed.

![](https://miro.medium.com/max/1400/0*8jjmsp_zOQfl0-Q8)
<small>Figure 6. *GP takes section data, uses a section component to turn it into UI (TitleSectionComponent from fig. 5), and presents the built section UI to the user.*</small>

## Screens

Screens are another building block of GP, but unlike sections, screens are mostly handled by GP client frameworks and are more opinionated in usage. GP screens are responsible for the layout and organization of sections.

### Screens schema

Screens are received as a `ScreenContainer` type. Screens can be launched in a modal (popup), in a bottom sheet, or as a full screen, depending on values included in the `screenProperties` field.

Screens enable dynamic configuration of a screen’s layout and, in turn, arrangement of sections through a `LayoutsPerFormFactor` type. `LayoutsPerFormFactor` specifies the layout for compact and wide breakpoints using an interface called `ILayout`, which will be elaborated on below. The GP framework on each client then uses screen density, rotation, and other factors to determine which `ILayout` from`LayoutsPerFormFactor` to render.

```graphql
type ScreenContainer {
  id: String
  
  # Properties such as how to launch this screen (popup, sheet, etc.)
  screenProperties: ScreenProperties
  
  layout: LayoutsPerFormFactor
}

# Specifies the ILayout type depending on rotation, client screen density, etc.
type LayoutsPerFormFactor {
  
  # Compact is usually used for portrait breakpoints (i.e. mobile phones)
  compact: ILayout
  
  # Wide is usually used for landscape breakpoints (i.e. web browsers, tablets)
  wide: ILayout
}
```
<small>Figure 7. A sample of the GP screens schema.</small>

### ILayouts

![](https://miro.medium.com/max/1400/0*vSJJB0hlGHrzSlu6)
<small>Figure 8. *A few examples of ILayout implementations, which are used to specify various placements.*</small>

`ILayout`s enable screens to change layouts depending on the response. In schema, `ILayout` is an interface with each `ILayout` implementation specifying various placements. Placements contain one or many `SectionDetail` types that point to sections in the response’s outermost `sections` array. We point to section data models rather than including them inline. This shrinks response sizes by reusing sections across layout configurations (`LayoutsPerFormFactor` from fig. 7).

```graphql
interface ILayout {}

type SectionDetail {
  # References a SectionContainer in the GPResponse.sections array
  sectionId: String
  
  # Styling data
  topPadding: Int
  bottomPadding: Int
  
  # ... Other styling data (margins, borders, etc)
}

# A placement meat to display a single GP section
type SingleSectionPlacement {
  sectionDetail: SectionDetail!
}

# A placement meat to display multiple GP sections in the order they appear in the sectionDetails array
type MultipleSectionsPlacement {
  sectionDetails: [SectionDetail]!
}

# A layout implementation defines the placements that sections are inserted into.
type SingleColumnLayout implements ILayout {
  
  nav: SingleSectionPlacement
  
  main: MultipleSectionsPlacement
  
  floatingFooter: SingleSectionPlacement
}
```
<small>Figure 9. *A sample of GP’s ILayout schema.*</small>

GP client frameworks inflate `ILayout`s for developers, as `ILayout` types are more opinionated than sections. Each`ILayout` has a unique renderer in each client’s GP framework. The layout renderer takes each `SectionDetail` from each placement, finds the proper section component to render that section, builds the section’s UI using that section component, and finally, places the built UI into the layout.

---

## Actions

The last concept of GP is our action and event handling infrastructure. One of the most game-changing aspects of GP is that in addition to defining the sections and layout of a screen from the network response, we also can define actions taken when users interact with UI on the screen, such as tapping a button or swiping a card. We do this through an `IAction` interface in our schema.

```graphql
interface IAction {}

# A simple action that will navigate the user to the screen matching the screenId when invoked
type NavigateToScreen implements IAction {
  screenId: String
}

# A sample TitleSection using an IAction type to handle the click of the subtitle
type TitleSection {
  ...
  
  subtitle: String
  
  # Action to be taken when tapping the subtitle
  onSubtitleClickAction: IAction
}
```
<small>*Figure 10. A sample of the GP IAction schema:*</small>

Recall from earlier (fig. 6) that a section component is what translates our `TitleSection` to UI on each client. Let’s take a look at the same Android example of a `TitleSectionComponent` with a dynamic `IAction` fired on the click of the subtitle text.

```kotlin
@SectionComponentType(SectionComponentType.TITLE)
class TitleSectionComponent : SectionComponent<TitleSection>() {
 
    override fun buildSectionUI(section: TitleSection) {

        // Build title UI elements

        if (!section.subtitle.isNullOrEmpty() {

            Text(
                ...
                onClick = {
                  GPActionHandler.handleIAction(section.onSubtitleClickAction)
                }
            )
        }
    }
}
```
<small>Figure 11. An example section component with an IAction fired on the click of a subtitle.</small>

When a user taps the subtitle in this section, it fires the `IAction` passed for the `onSubtitleClickAction` field in `TitleSection`. GP is responsible for routing this action to an event handler defined for the feature, which will handle the `IAction` that was fired.

There is a standard set of generic actions that GP handles universally, such as navigating to a screen or scrolling to a section. Features can add their own `IAction` types and use those to handle their feature’s unique actions. Since the feature-specific event handlers are scoped to the feature, they can contain as much feature-specific business logic as they wish, enabling freedom to use custom actions and business logic when specific use cases arise.

---

## Bringing It All Together

We’ve gone over several concepts, so let’s take an entire GP response and see how it’s rendered to tie everything together.

```json
{
  "screens": [
    {
      "id": "ROOT",
      "screenProperties": {},
      "layout": {
        "wide": {},
        "compact": {
          "type": "SingleColumnLayout",
          "main": {
            "type": "MultipleSectionsPlacement",
            "sectionDetails": [
              {
                "sectionId": "hero_section"
              },
              {
                "sectionId": "title_section"
              }
            ]
          },
          "nav": {
            "type": "SingleSectionPlacement",
            "sectionDetail": {
              "sectionId": "toolbar_section"
            }
          },
          "footer": {
            "type": "SingleSectionPlacement",
            "sectionDetail": {
              "sectionId": "book_bar_footer"
            }
          }
        }
      }
    },
  ],
  "sections": [
    {
      "id": "toolbar_section",
      "sectionComponentType": "TOOLBAR",
      "section": {
        "type": "ToolbarSection",
        "nav_button": {
          "onClickAction": {
            "type": "NavigateBack",
            "screenId": "previous_screen_id"
          }
        }
      }
    },
    {
      "id": "hero_section",
      "sectionComponentType": "HERO",
      "section": { 
        "type": "HeroSection",
        "images": [
          "api.airbnb.com/...",
        ],
      }
    },
    {
      "id": "title_section",
      "sectionComponentType": "TITLE",
      "section": { 
        "type": "TitleSection",
        "title": "Seamist Beach Cottage, Private Beach & Ocean Views",
        "titleStyle": {}
      }
    },
    {
      "id": "book_bar_footer",
      "sectionComponentType": "BOOK_BAR_FOOTER",
      "section": {
        "type": "ButtonSection", 
        "title": "$450/night",
        "button": {
          "text": "Check Availability",
          "onClickAction": {
            "type": "NavigateToScreen",
            "screenId": "next_screen_id"
          }
        },
      }
    },
  ]
}
```
<small>Figure 12. An example JSON of a valid GP response.</small>

### Creating the Section Components

Features using GP will need to fetch a response implementing `GPResponse` mentioned above (fig. 2). Upon receiving a `GPResponse`, GP infra handles parsing this response and building the sections for the developer.

Recall that each section in our `sections` array has a `SectionComponentType` and a `section` data model. Developers working on GP add section components, using `SectionComponentType` as the key for how to render the section data model.

GP finds each section component and passes it the corresponding data model. Each section component creates UI components for the section, which GP will insert into the proper placement in the layout below.

![](https://miro.medium.com/max/1400/0*2SLVYciGoKNbdau6)

<small>Figure 13. *Transforming section data to UI.*</small>

### Handling Actions

Now that each section component’s UI elements are set up, we need to handle users interacting with the sections. For example, if they tap a button, we need to handle the action taken on click.

Recall earlier that GP handles routing events to their proper handler. The example response above (fig. 12) contains two sections that can fire actions, `toolbar_section` and `book_bar_footer`. The section component for building both of these sections simply needs to take the `IAction` and specify when to fire it, which in both cases will be when a button is clicked.

We can do this through click handlers on each client, which will use GP infra to route the event on a click event.

```graphql
button(  
  onClickListener = {  
    GPActionHandler.handleIAction(section.button.onClickAction)  
  }  
)
```

### Setting Up the Screen and Layout

To arrange a fully interactive screen for our users, GP looks through the screens array to find a screen with the `“ROOT”` id (GP’s default screen id). GP will then find the proper `ILayout` type depending on the breakpoint and other factors relevant to the specific device the user is using. To keep things simple, we’ll use the layout from the `compact` field, a `SingleColumnLayout`.

GP will then find a layout renderer for `SingleColumnLayout`, where it’ll inflate a layout with a top container (the `nav` placement), a scrollable list (the `main` placement), and a floating footer (the `footer` placement).

This layout renderer will take the models for the placements, which contain `SectionDetail` objects. These `SectionDetail`s contain some styling information as well as the `sectionId` of the section to inflate. GP will iterate through these `SectionDetail` objects and inflate sections into their respective placements using the section components we built earlier.

![](https://miro.medium.com/max/1400/0*8mJWV_F1FPq0U5eI)
<small>Figure 14. *GP Infra takes built sections with action handlers added, adds sections to ILayout placements.*</small>

## What’s Next for GP?

GP has only existed for about a year, but a majority of Airbnb’s most used features (e.g., search, listing pages, checkout) are built on GP. Despite the critical mass of usage, GP is still in its infancy and there’s much more to be done.

We have plans for a more composable UI through “nested sections”, improving discoverability of elements that already exist through our design tools, such as Figma, and WYSIWYG editing of sections and placements, enabling no-code feature changes.

If you’re passionate about server-driven UI or building UI systems that scale, there’s so much more to be done. We encourage you to apply to the [open roles](https://careers.airbnb.com/) on our engineering team.

### Re-engineering Travel Tech Talk

Server-driven UI is complex. Countless hours have gone into creating a robust schema, client frameworks, and developer documentation that enables GP to be successful.

If you’d like a more high-level overview of SDUI and GP, I recently had the opportunity to speak at Airbnb’s [Re-engineering Travel tech talk](https://www.facebook.com/AirbnbTech/videos/1445539065813160/) presenting GP. I’d encourage you to check it out for a general overview of server-driven UI and GP (skip to the 31-minute mark if you’re short on time).

### Special Thanks

Special thanks to [Abhi Vohra](https://www.linkedin.com/in/abhinavvohra/), [Wensheng Mao](https://www.linkedin.com/in/wensheng-mao-76ab7142/), [Jean-Nicolas Vollmer](https://www.linkedin.com/in/jnvollmer/), [Pranay Airan](https://www.linkedin.com/in/pranayairan/), [Stephen Herring](https://www.linkedin.com/in/stephen-herring-00381a6a/), [Jasper Liu](https://www.linkedin.com/in/jsperl/), [Kevin Weber](https://www.linkedin.com/in/kevinchrisweber/), [Rodolphe Courtier](https://www.linkedin.com/in/rodolphe-courtier-97b32610/), [Daniel Garcia-Carrillo](https://www.linkedin.com/in/danielgarciacarrillo/), [Fidel Sosa](https://www.linkedin.com/in/fidelsosa/), [Roshan Goli](https://www.linkedin.com/in/roshan-goli-03977a25/), [Cal Stephens](https://www.linkedin.com/in/calstephens/), [Chen Wu](https://www.linkedin.com/in/chen-wu-a5677649/), [Nick Miller](https://www.linkedin.com/in/nickbryanmiller/), [Yanlin Chen](https://www.linkedin.com/in/yanlin-chen-58a41411/), [Susan Dang](https://www.linkedin.com/in/rsusandang/), and [Amity Wang](https://www.linkedin.com/in/amitywang/), as well as many more behind the scenes, for their tireless work building and supporting GP.

> 如果发现译文存在错误或其他需要改进的地方，欢迎到 [掘金翻译计划](https://github.com/xitu/gold-miner) 对译文进行修改并 PR，也可获得相应奖励积分。文章开头的 **本文永久链接** 即为本文在 GitHub 上的 MarkDown 链接。

---

> [掘金翻译计划](https://github.com/xitu/gold-miner) 是一个翻译优质互联网技术文章的社区，文章来源为 [掘金](https://juejin.im) 上的英文分享文章。内容覆盖 [Android](https://github.com/xitu/gold-miner#android)、[iOS](https://github.com/xitu/gold-miner#ios)、[前端](https://github.com/xitu/gold-miner#前端)、[后端](https://github.com/xitu/gold-miner#后端)、[区块链](https://github.com/xitu/gold-miner#区块链)、[产品](https://github.com/xitu/gold-miner#产品)、[设计](https://github.com/xitu/gold-miner#设计)、[人工智能](https://github.com/xitu/gold-miner#人工智能)等领域，想要查看更多优质译文请持续关注 [掘金翻译计划](https://github.com/xitu/gold-miner)、[官方微博](http://weibo.com/juejinfanyi)、[知乎专栏](https://zhuanlan.zhihu.com/juejinfanyi)。
