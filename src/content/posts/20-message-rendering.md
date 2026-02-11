---
title: "Message Rendering: From Model to Pixels"
description: "How a Telegram message goes from a data model to a rendered bubble on screen. Covers ChatMessageItemImpl and the content routing system that selects from 20+ node types, ChatMessageBubbleItemNode's 7,236-line layout engine with its multi-container node tree, the bubble content protocol with four-position sizing, message merging rules, async two-phase layout, date headers with sticky positioning, and the complete pipeline from Message to CALayer."
published: 2025-07-29
tags:
  - Feature Deep Dives
toc: true
lang: en
abbrlink: 20-message-rendering
pin: 6
---

Every message you see in a Telegram chat — whether it's a text bubble, a photo, a sticker, a voice note, or a poll — passes through a sophisticated rendering pipeline that transforms a `Message` data model into positioned, themed, interactive pixels. This pipeline must be fast (60fps scrolling with thousands of messages), flexible (20+ content types with different layouts), and correct (proper bubble corners, merge groups, date headers, and read states).

This post traces the complete journey from data model to rendered node.

## Step 1: Message → ChatMessageItem

The bridge between the data layer and the UI layer is the `ChatMessageItem` protocol, defined in its own module:

```swift
public protocol ChatMessageItem: ListViewItem {
    var presentationData: ChatPresentationData { get }
    var context: AccountContext { get }
    var chatLocation: ChatLocation { get }
    var associatedData: ChatMessageItemAssociatedData { get }
    var controllerInteraction: ChatControllerInteraction { get }
    var content: ChatMessageItemContent { get }
    var headers: [ListViewItemHeader] { get }
    var message: Message { get }
    var read: Bool { get }
    var unsent: Bool { get }
    var sending: Bool { get }
    var failed: Bool { get }
}
```

The `content` property captures whether this is a single message or a group of messages that should be displayed together (photo albums):

```swift
public enum ChatMessageItemContent: Sequence {
    case message(message: Message, read: Bool, selection: ..., location: ...)
    case group(messages: [(Message, Bool, ...)])
}
```

The concrete implementation is `ChatMessageItemImpl` (675 lines). When the history view produces messages, each one is wrapped into a `ChatMessageItemImpl` that:

1. Computes `displayAuthorInfo` — should we show the sender's name and avatar?
2. Creates date headers from the message timestamp
3. Determines the merge status with neighboring messages
4. Stores the effective author (important for channels with post signatures)

## Step 2: Node Class Selection

When a `ChatMessageItemImpl` needs to be rendered, `nodeConfiguredForParams()` selects the appropriate node class. This is the routing decision — the point where the system decides how to display the message:

```swift
var viewClassName: AnyClass = ChatMessageBubbleItemNode.self

for media in self.message.media {
    if let telegramFile = media as? TelegramMediaFile {
        if telegramFile.isVideoSticker || telegramFile.isAnimatedSticker {
            viewClassName = ChatMessageAnimatedStickerItemNode.self
            break
        }
        for attribute in telegramFile.attributes {
            if case .Sticker = attribute {
                viewClassName = ChatMessageStickerItemNode.self
                break
            }
        }
    } else if media is TelegramMediaAction {
        viewClassName = ChatMessageBubbleItemNode.self
    }
}

// Large emoji override
if viewClassName == ChatMessageBubbleItemNode.self && self.presentationData.largeEmoji {
    switch attributes.contentTypeHint {
    case .largeEmoji:
        viewClassName = ChatMessageStickerItemNode.self
    case .animatedEmoji:
        viewClassName = ChatMessageAnimatedStickerItemNode.self
    }
}
```

Three top-level node types handle the vast majority of messages:

| Node Class | Used For |
|------------|----------|
| `ChatMessageBubbleItemNode` | Text, photos, videos, files, links, polls, contacts, locations, games, invoices, actions |
| `ChatMessageStickerItemNode` | Static stickers, large emoji |
| `ChatMessageAnimatedStickerItemNode` | Animated stickers, video stickers, animated emoji |

The bubble node is the general-purpose container. Stickers and animated stickers get their own node classes because they don't have bubble backgrounds — they render directly onto the chat wallpaper.

## Step 3: Content Node Routing (Inside the Bubble)

Once `ChatMessageBubbleItemNode` is selected, a second level of routing determines which **content nodes** fill the bubble. The function `contentNodeMessagesAndClassesForItem` (113 lines) maps media types to content node classes:

```swift
func contentNodeMessagesAndClassesForItem(_ item: ChatMessageItem)
    -> ([(Message, AnyClass, ChatMessageEntryAttributes, BubbleItemAttributes)],
        Bool, Bool)
```

The routing table:

| Media Type | Content Node |
|------------|-------------|
| `TelegramMediaImage` | `ChatMessageMediaBubbleContentNode` |
| `TelegramMediaFile` (video) | `ChatMessageMediaBubbleContentNode` |
| `TelegramMediaFile` (audio/voice) | `ChatMessageFileBubbleContentNode` |
| `TelegramMediaFile` (instant round video) | `ChatMessageInstantVideoBubbleContentNode` |
| `TelegramMediaWebpage` | `ChatMessageWebpageBubbleContentNode` |
| `TelegramMediaPoll` | `ChatMessagePollBubbleContentNode` |
| `TelegramMediaContact` | `ChatMessageContactBubbleContentNode` |
| `TelegramMediaMap` | `ChatMessageMapBubbleContentNode` |
| `TelegramMediaGame` | `ChatMessageGameBubbleContentNode` |
| `TelegramMediaInvoice` | `ChatMessageInvoiceBubbleContentNode` |
| `TelegramMediaAction` (call) | `ChatMessageCallBubbleContentNode` |
| `TelegramMediaAction` (gift) | `ChatMessageGiftBubbleContentNode` |
| `TelegramMediaAction` (wallpaper) | `ChatMessageWallpaperBubbleContentNode` |
| `TelegramMediaTodo` | `ChatMessageTodoBubbleContentNode` |
| Text only | `ChatMessageTextBubbleContentNode` |

A single message can contain **multiple content nodes**. For example, a text message with an image gets both `ChatMessageTextBubbleContentNode` and `ChatMessageMediaBubbleContentNode`. The order depends on the `InvertMediaMessageAttribute` — if present, text appears above the media instead of below.

## Step 4: The Bubble Item Node

`ChatMessageBubbleItemNode` at **7,236 lines** is the workhorse of message rendering. It manages a complex node tree:

```
ChatMessageBubbleItemNode (extends ChatMessageItemView)
├── mainContextSourceNode (ContextExtractedContentContainingNode)
│   └── mainContainerNode (ContextControllerSourceNode)
│       ├── backgroundWallpaperNode (ChatMessageBubbleBackdrop)
│       ├── backgroundNode (ChatMessageBackground)
│       └── clippingNode (ChatMessageBubbleClippingNode)
│           └── contentContainersWrapperNode
│               └── ContentContainer[0..n]
│                   ├── sourceNode
│                   └── containerNode
│                       ├── backgroundWallpaperNode
│                       └── backgroundNode
├── nameNode (TextNode — author name)
├── forwardInfoNode (ChatMessageForwardInfoNode)
├── replyInfoNode (ChatMessageReplyInfoNode)
├── contentNodes: [ChatMessageBubbleContentNode]
├── actionButtonsNode (ChatMessageActionButtonsNode)
├── reactionButtonsNode (ChatMessageReactionButtonsNode)
├── selectionNode (ChatMessageSelectionNode)
├── deliveryFailedNode (ChatMessageDeliveryFailedNode)
├── swipeToReplyNode (ChatMessageSwipeToReplyNode)
└── messageAccessibilityArea (AccessibilityAreaNode)
```

### Why Content Containers?

Grouped messages (photo albums) use multiple `ContentContainer` instances — one per message in the group. Each container has its own background node so that context menus can extract individual messages from the group. For single messages, there's just one container.

### The Context Source Nodes

The `ContextExtractedContentContainingNode` and `ContextControllerSourceNode` are part of Telegram's custom context menu system. When the user long-presses a message, the context source node "extracts" the bubble from the list and presents it above a blurred backdrop with action buttons. This is why the node hierarchy wraps everything in context-aware containers.

## Step 5: The Bubble Content Protocol

All content nodes inherit from `ChatMessageBubbleContentNode`:

```swift
open class ChatMessageBubbleContentNode: ASDisplayNode {
    open var supportsMosaic: Bool { return false }
    public weak var itemNode: ChatMessageItemNodeProtocol?
    public weak var bubbleBackgroundNode: ChatMessageBackground?
    open var visibility: ListViewItemNodeVisibility = .none

    open func asyncLayoutContent()
        -> (_ item: ChatMessageBubbleContentItem,
            _ layoutConstants: ChatMessageItemLayoutConstants,
            _ preparePosition: ChatMessageBubblePreparePosition,
            _ messageSelection: Bool?,
            _ constrainedSize: CGSize,
            _ avatarInset: CGFloat)
        -> (ChatMessageBubbleContentProperties,
            unboundSize: CGSize?,
            maxWidth: CGFloat,
            layout: (CGSize, ChatMessageBubbleContentPosition) -> (...))
}
```

The `asyncLayoutContent()` method returns a **curried function** — a layout closure that takes parameters in stages:

1. **First call**: receives the item, constants, and constraints → returns properties, unbound size, and max width
2. **Second call** (the inner `layout` closure): receives the final size and position → returns the actual node layout

This two-phase design allows the bubble node to measure all content nodes first (phase 1), compute the bubble's total size, then position everything (phase 2). The `supportsMosaic` property indicates whether the content node can participate in photo grid layouts (image and video nodes return `true`).

## Step 6: Layout Constants

Layout dimensions adapt to the screen width through `ChatMessageItemLayoutConstants`:

```swift
public struct ChatMessageItemLayoutConstants {
    public var avatarInset: CGFloat = 38.0
    public var timestampHeaderHeight: CGFloat = 34.0

    public var bubble: ChatMessageItemBubbleLayoutConstants
    // edgeInset, defaultSpacing, mergedSpacing, maximumWidthFill, minimumSize

    public var image: ChatMessageItemImageLayoutConstants
    // maxDimensions: 300-440 × 380-440, minDimensions: 170 × 74
    // cornerRadius: 16.0 (standalone), 8.0 (merged)

    public var text: ChatMessageItemTextLayoutConstants
    // bubbleInsets: top 6, left 10-11, bottom 6, right 10-11

    public var file: ChatMessageItemFileLayoutConstants
    // bubbleInsets: top 15, left 9, bottom 15, right 12

    public var instantVideo: ChatMessageItemInstantVideoConstants
    // dimensions: 212×212 (compact) or 240×240 (regular)
}
```

On compact screens (iPhone SE), images max out at 300×380 points. On regular-width layouts (iPad), they expand to 440×440. Text bubbles use 10-11 points of horizontal padding. File bubbles use slightly more vertical padding (15 points) to accommodate the file icon and download progress indicator.

## Step 7: Bubble Positioning

Each content node within a bubble receives a `ChatMessageBubbleContentPosition` that describes its relationship to neighboring content:

```swift
public enum ChatMessageBubbleRelativePosition {
    public enum NeighbourType {
        case media
        case header
        case footer
        case text
        case reactions
    }

    public enum NeighbourSpacing {
        case `default`
        case condensed
        case overlap(CGFloat)  // Negative spacing (e.g., -4.0 for files)
    }

    case None(ChatMessageBubbleMergeStatus)
    case BubbleNeighbour
    case Neighbour(Bool, NeighbourType, NeighbourSpacing)
}
```

The `overlap` spacing is used for file messages — when multiple files are stacked, they overlap by 4-14 points so their separators sit right against each other without extra gaps.

Merge status controls the bubble's corner rounding:

```swift
public enum ChatMessageBubbleMergeStatus {
    case None(ChatMessageBubbleNoneMergeStatus)  // Standalone
    case Left   // Top of merged group
    case Right  // Bottom of merged group
    case Both   // Middle of merged group
}
```

When two messages merge (same author, within 10 minutes), the bottom message gets `.Left` status (round top-right, square top-left for incoming), creating the characteristic Telegram grouped-bubble appearance.

## Step 8: Message Merging

The function `messagesShouldBeMerged` determines whether consecutive messages should visually connect:

```swift
func messagesShouldBeMerged(message: Message, previous: Message)
    -> ChatMessageMerge
```

Merge conditions (all must be true):
1. Timestamps within **10 minutes** of each other
2. Same chat (peer ID matches)
3. Same author and same direction (incoming/outgoing)
4. Same reply thread
5. Neither message is a service action (join, leave, photo change)
6. No expired media

The result is a three-value enum:

```swift
public enum ChatMessageMerge: Int32 {
    case none = 0              // Distinct bubble with rounded corners
    case fullyMerged = 1       // Connected bubble (squared corner on merge side)
    case semanticallyMerged = 2 // Grouped but visually distinct (stickers)
}
```

## Step 9: Async Two-Phase Layout

The bubble node's `asyncLayout()` method returns a closure chain that executes the two-phase layout:

**Phase 1 — Measurement** (runs on a background thread):
- Compute the maximum content width from constraints
- Layout each content node via `asyncLayoutContent()` to get unbound sizes
- Measure the author name, forward info, and reply info text
- Determine the bubble width (maximum of all content widths)
- Calculate the status bar size (date, checkmarks, reaction count)

**Phase 2 — Finalization** (can run on any thread):
- Apply the computed bubble size to background nodes
- Position content nodes vertically within the bubble
- Set corner radii based on merge status
- Position the author name, forward info, and reply info
- Wire up the action buttons and reaction buttons below the bubble

This two-phase approach means the expensive text measurement and image sizing happens off the main thread, while the final positioning (which needs access to the node hierarchy) can be applied efficiently.

## Step 10: Text Content Rendering

`ChatMessageTextBubbleContentNode` handles the most common content type — text messages:

```swift
public class ChatMessageTextBubbleContentNode: ChatMessageBubbleContentNode {
    private let containerNode: ContainerNode
    private let textNode: InteractiveTextNodeWithEntities
    public var statusNode: ChatMessageDateAndStatusNode?
    private var linkHighlightingNode: LinkHighlightingNode?
    private var textSelectionNode: TextSelectionNode?
}
```

The text layout pipeline:

1. **Extract text and entities** from the message
2. **Apply formatting** — bold, italic, code, links, mentions, hashtags, spoilers, block quotes
3. **Handle code blocks** with syntax highlighting (cached for performance)
4. **Apply theme colors** — text color, link color, code background
5. **Measure with CoreText** via `InteractiveTextNodeWithEntities`
6. **Calculate status width** — the date stamp and checkmarks must fit on the last line or overflow to a new line
7. **Return the frame** with proper text insets (2pt padding)

Spoiler text is rendered as shimmer-animated rectangles that reveal on tap. Block quotes get a tinted background with a left border. Code blocks get a monospace font and a dark/light background.

## Step 11: Media Content Rendering

`ChatMessageMediaBubbleContentNode` handles images and videos:

```swift
public class ChatMessageMediaBubbleContentNode: ChatMessageBubbleContentNode {
    override public var supportsMosaic: Bool { return true }
    private let interactiveImageNode: ChatMessageInteractiveMediaNode
    private var selectionNode: GridMessageSelectionNode?
}
```

The `supportsMosaic` property is `true`, meaning this node can participate in photo album grid layouts. The underlying `ChatMessageInteractiveMediaNode` (~215KB) is one of the largest files in the codebase, handling:

- Aspect-fill image display with progressive loading
- Video player embedding with autoplay
- Download progress indication
- Age-restricted content with blur overlay and reveal animation
- Gesture recognition (tap to open, pinch to zoom, long-press for context menu)
- Media transitions for smooth gallery open/close animations

### Autoplay Logic

Media nodes decide whether to auto-download and auto-play based on:
- Network type (WiFi vs cellular)
- User's auto-download settings
- Whether the file is already cached
- Battery saving mode
- Energy impact settings

GIFs auto-play when fully downloaded. Videos only auto-play when scrolled into view, and only if the user's settings allow it.

## Step 12: Date Headers

Date headers are sticky section headers that separate messages by day:

```swift
public final class ChatMessageDateHeader: ListViewItemHeader {
    public struct Id: Hashable {
        public let roundedTimestamp: Int64?
        public let separableThreadId: Int64?
    }

    public let stickDirection: ListViewItemHeaderStickDirection
    public let height: CGFloat = 34.0
}
```

Each message's timestamp is rounded to a day boundary. Messages on the same day share the same header ID, so the list view displays one header per day. The `stickDirection` is `.bottom` in the default rotated chat (newest at bottom), meaning headers stick to the top of the screen as you scroll up through older messages.

The header renders as a rounded capsule with the date text centered inside, using the theme's service message colors. In forum topics, an additional stacking header shows the topic name above the date.

## Step 13: Bubble Backgrounds

The bubble background is drawn by `ChatMessageBackground`, which selects colors from the current theme:

```swift
let messageTheme = incoming ?
    presentationData.theme.theme.chat.message.incoming :
    presentationData.theme.theme.chat.message.outgoing
let bubbleColor = graphics.hasWallpaper ?
    messageTheme.bubble.withWallpaper.fill :
    messageTheme.bubble.withoutWallpaper.fill
```

Each bubble has two color variants — one for wallpaper backgrounds and one for plain backgrounds. This dual-color system ensures bubbles are readable against both dark wallpapers and light solid backgrounds.

The bubble corners use precomposited images — rather than applying `cornerRadius` to each bubble (which triggers expensive GPU compositing), Telegram generates corner images at theme creation time and uses `UIImage.resizableImage(withCapInsets:)` to stretch them efficiently.

## The Complete Pipeline

```
Message (TelegramCore data model)
    ↓
ChatMessageItemImpl (wraps message + merge status + headers)
    ↓
nodeConfiguredForParams() — selects node class
    ├── ChatMessageBubbleItemNode (text, media, files, etc.)
    ├── ChatMessageStickerItemNode (stickers)
    └── ChatMessageAnimatedStickerItemNode (animated stickers)
    ↓
contentNodeMessagesAndClassesForItem() — selects content nodes
    ├── ChatMessageTextBubbleContentNode
    ├── ChatMessageMediaBubbleContentNode
    ├── ChatMessageFileBubbleContentNode
    └── ... (20+ content types)
    ↓
asyncLayout() Phase 1 — measure all components (background thread)
    ├── Text node: CoreText measurement
    ├── Media node: aspect ratio calculation
    ├── Status node: date + checkmarks width
    ├── Author name: text measurement
    └── Reply info: quoted message measurement
    ↓
asyncLayout() Phase 2 — position nodes (apply closure)
    ├── Bubble background: sized and cornered
    ├── Content nodes: stacked vertically
    ├── Status node: bottom-right of bubble
    ├── Reaction buttons: below bubble
    └── Action buttons: below reactions
    ↓
CADisplayLink — render to screen at 60/120fps
```

This pipeline processes a simple text message in under a millisecond and a complex media message in 2-3ms. The key insight is that the expensive work (text measurement, image sizing) happens off the main thread in Phase 1, while Phase 2 only does simple arithmetic positioning. Combined with AsyncDisplayKit's display sentinel system (which cancels renders for off-screen cells), this keeps scrolling buttery smooth even in chats with thousands of messages mixing text, images, videos, and interactive content.
