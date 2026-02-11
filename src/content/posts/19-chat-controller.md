---
title: "ChatController: The Heart of the Messaging Experience"
description: "A deep dive into ChatControllerImpl — at 10,413 lines, Telegram iOS's most complex view controller. Covers the 130-import dependency graph, 50+ disposable properties, the ChatControllerInteraction closure bag with 80+ callbacks, ChatControllerNode's layered display hierarchy, the ContentData reactive state pipeline, input panel management, and how the chat screen coordinates messages, media, recording, search, and overlays into a seamless experience."
published: 2025-07-28
tags:
  - Feature Deep Dives
toc: true
lang: en
abbrlink: 19-chat-controller
pin: 7
---

Open any conversation in Telegram. What you see — the message list, the input bar, the navigation title, the wallpaper, the typing indicators, the pinned message banner, the reactions floating past — is all driven by a single view controller: `ChatControllerImpl`. At **10,413 lines** in `ChatController.swift` alone (with another **5,114 lines** in `ChatControllerNode.swift`), it is by far the largest and most complex component in the entire codebase.

This post dissects how it works.

## The Scale of the Problem

`ChatControllerImpl` imports **130 modules** — more than half the app's entire module graph. The first 149 lines of the file are nothing but `import` statements:

```swift
import TelegramCore
import TelegramPresentationData
import AsyncDisplayKit
import Display
import ChatControllerInteraction
import ChatMessageBubbleItemNode
import ChatMessageItemImpl
import EntityKeyboard
import AttachmentUI
import MediaPickerUI
import StoryContainerScreen
import VideoMessageCameraScreen
import TranslateUI
import PremiumUI
// ... 116 more imports
```

Why so many? Because the chat screen is the convergence point of almost every feature in the app. It must handle text messages, photos, videos, stickers, GIFs, voice messages, video messages, polls, locations, contacts, games, invoices, payments, web apps, reactions, replies, forwards, edits, translations, scheduled messages, slow mode, secret chats, group calls, stories, ads, bots, inline queries, search, themes, wallpapers — all in one screen. Every one of those features lives in its own module, and ChatController must orchestrate them all.

## Class Declaration and Key Properties

```swift
public final class ChatControllerImpl: TelegramBaseController, ChatController,
    GalleryHiddenMediaTarget, UIDropInteractionDelegate
{
    let context: AccountContext
    public internal(set) var chatLocation: ChatLocation
    public internal(set) var subject: ChatControllerSubject?
    var mode: ChatControllerPresentationMode

    var presentationInterfaceState: ChatPresentationInterfaceState
    let presentationInterfaceStatePromise: ValuePromise<ChatPresentationInterfaceState>

    public private(set) var controllerInteraction: ChatControllerInteraction?
    var interfaceInteraction: ChatPanelInterfaceInteraction?

    let chatBackgroundNode: WallpaperBackgroundNode
    // ...
}
```

`ChatControllerImpl` inherits from `TelegramBaseController` (which itself inherits from `ViewController`, Telegram's custom base class built on `ASDisplayNode`). It conforms to the `ChatController` protocol — the public interface that other modules use to create and interact with chat screens.

### ChatLocation: What Chat Are We In?

The `chatLocation` property determines what conversation is displayed:

```swift
public enum ChatLocation: Equatable {
    case peer(id: PeerId)
    case replyThread(message: ChatReplyThreadMessage)
    case customChatContents
}
```

A `peer` is the simplest case — a 1:1 chat, group, or channel identified by `PeerId`. A `replyThread` represents a comment thread under a channel post or a forum topic. `customChatContents` is used for special screens like quick-reply message editing and business link setup, where the chat UI is repurposed as a general message composer.

### Presentation Mode

The `mode` property controls how the chat is presented:

```swift
public enum ChatControllerPresentationMode {
    case standard(StandardMode)
    case overlay
    case inline(NavigationBarPresentationData?)

    public enum StandardMode {
        case `default`
        case embedded
        case previewing
    }
}
```

Most chats use `.standard(.default)`. The `.overlay` mode is used for picture-in-picture-style chat bubbles. `.inline` is for embedded chat views within other screens. `.embedded` is used when a chat is shown inside another controller (like the saved messages sub-chat list). Each mode changes the status bar style, navigation bar visibility, and gesture handling.

## The 50+ Disposable Properties

One of the most striking things about `ChatControllerImpl` is its roster of disposable properties — over **50** of them:

```swift
let navigationActionDisposable = MetaDisposable()
let messageIndexDisposable = MetaDisposable()
var networkStateDisposable: Disposable?
let sentMessageEventsDisposable = MetaDisposable()
let failedMessageEventsDisposable = MetaDisposable()
let messageActionCallbackDisposable = MetaDisposable()
let editMessageDisposable = MetaDisposable()
let enqueueMediaMessageDisposable = MetaDisposable()
var audioRecorderDisposable: Disposable?
var videoRecorderDisposable: Disposable?
var chatUnreadCountDisposable: Disposable?
var peerInputActivitiesDisposable: Disposable?
var translationStateDisposable: Disposable?
var powerSavingMonitoringDisposable: Disposable?
// ... and 35+ more
```

Each disposable represents an active signal subscription — a stream of data from TelegramCore, Postbox, or the network. The chat screen simultaneously observes:

- The peer's profile data and online status
- Unread message counts and mention counts
- Typing activity from other participants
- Network connectivity state
- Media download settings changes
- Sticker and emoji settings changes
- Translation state
- Power saving mode
- Auto-night mode
- Active group call state
- Send-as peer options
- Slowmode cooldown timers
- Pinned messages
- Ad messages

All of these run concurrently and feed into the presentation state, which then drives a single coordinated UI update.

## The Presentation Interface State

The single most important data structure in the chat screen is `ChatPresentationInterfaceState`. This immutable value type captures the **complete** UI state of the chat:

```swift
var presentationInterfaceState: ChatPresentationInterfaceState
let presentationInterfaceStatePromise: ValuePromise<ChatPresentationInterfaceState>
```

When anything changes — the user types text, a pinned message appears, the peer goes online, the keyboard opens — the controller calls `updateChatPresentationInterfaceState`:

```swift
self.updateChatPresentationInterfaceState(
    transition: .animated(duration: 0.4, curve: .spring),
    interactive: false,
    force: true
) { presentationInterfaceState in
    var state = presentationInterfaceState
    state = state.updatedPeer({ _ in contentData.state.renderedPeer })
    state = state.updatedContactStatus(contentData.state.contactStatus)
    state = state.updatedHasBots(contentData.state.hasBots)
    state = state.updatedHasBotCommands(contentData.state.hasBotCommands)
    state = state.updatedIsNotAccessible(contentData.state.isNotAccessible)
    // ... many more updates
    return state
}
```

This pattern — take the current state, produce a new state via `updatedXyz()` methods, and return it — is used everywhere in the controller. The new state is then diffed against the old state, and only the changed parts trigger UI updates. The `transition` parameter controls whether changes animate (with spring curves) or apply immediately.

## Initialization: Building the Chat

The initializer at line 633 takes a rich set of parameters:

```swift
public init(
    context: AccountContext,
    chatLocation: ChatLocation,
    chatLocationContextHolder: Atomic<ChatLocationContextHolder?>,
    subject: ChatControllerSubject? = nil,
    botStart: ChatControllerInitialBotStart? = nil,
    attachBotStart: ChatControllerInitialAttachBotStart? = nil,
    botAppStart: ChatControllerInitialBotAppStart? = nil,
    mode: ChatControllerPresentationMode = .standard(.default),
    peekData: ChatPeekTimeout? = nil,
    chatListFilter: Int32? = nil,
    chatNavigationStack: [ChatNavigationStackItem] = [],
    customChatNavigationStack: [EnginePeer.Id]? = nil,
    params: ChatControllerParams? = nil
)
```

The initializer:

1. **Creates the wallpaper background node** — the animated wallpaper that sits behind all messages
2. **Loads presentation data** — theme, strings, font size, bubble corners
3. **Constructs the initial `ChatPresentationInterfaceState`** — a single massive struct combining chat location, theme, peer data, mode, and all UI flags
4. **Sets up the navigation bar** with glass-style presentation data
5. **Wires up scroll-to-top** and **attempt-navigation** closures for the navigation controller

The actual node hierarchy is created later, in `loadDisplayNode()`, following the standard Telegram pattern where the display node is lazily constructed when the view is first needed.

## ChatControllerNode: The Display Hierarchy

`ChatControllerNode` manages the visual layout of the chat screen. At line 165 of `ChatControllerNode.swift`:

```swift
class ChatControllerNode: ASDisplayNode, ASScrollViewDelegate {
    let context: AccountContext
    private(set) var chatLocation: ChatLocation
    let controllerInteraction: ChatControllerInteraction
    private weak var controller: ChatControllerImpl?

    let wrappingNode: SpaceWarpNode
    let contentContainerNode: ChatNodeContainer
    let backgroundNode: WallpaperBackgroundNode
    var historyNode: ChatHistoryListNodeImpl
    let historyNodeContainer: HistoryNodeContainer

    let inputPanelContainerNode: ChatInputPanelContainer
    let inputPanelBackgroundNode: NavigationBackgroundNode
    private(set) var inputPanelNode: ChatInputPanelNode?
    private(set) var accessoryPanelNode: AccessoryPanelNode?
    private(set) var inputNode: ChatInputNode?
    private(set) var textInputPanelNode: ChatTextInputPanelNode?

    let navigateButtons: ChatHistoryNavigationButtons
    var messageTransitionNode: ChatMessageTransitionNodeImpl
    // ...
}
```

The node hierarchy (simplified):

```
ChatControllerNode
├── wrappingNode (SpaceWarpNode — warp effect for message sends)
│   └── contentContainerNode
│       ├── backgroundNode (WallpaperBackgroundNode — animated wallpaper)
│       ├── historyNodeContainer
│       │   └── historyNode (ChatHistoryListNodeImpl — the message list)
│       ├── loadingPlaceholderNode (skeleton during initial load)
│       ├── emptyNode (shown when chat has no messages)
│       ├── inputPanelContainerNode
│       │   ├── inputPanelBackgroundNode (blurred glass bar)
│       │   ├── inputPanelNode (text/bot/restricted panel)
│       │   └── accessoryPanelNode (reply/forward/edit bar)
│       ├── titleAccessoryPanelContainer (pinned message, translate bar)
│       ├── headerPanelsView (ad panel, media playback, group call)
│       ├── navigateButtons (scroll-to-bottom, mention jump)
│       └── messageTransitionNode (send/receive animations)
├── inputContextPanelContainer (autocomplete: @mentions, /commands)
└── inputNode (full keyboard: emoji, stickers, media picker)
```

### The History Node Container and Secret Chats

The `HistoryNodeContainer` has a special property for secret chats:

```swift
class HistoryNodeContainer: ASDisplayNode {
    var isSecret: Bool {
        didSet {
            if self.isSecret != oldValue {
                setLayerDisableScreenshots(self.layer, self.isSecret)
            }
        }
    }
}
```

When the chat is a secret chat, the history container uses iOS's `DRM` layer flag to prevent screen recording and screenshots. This is the same mechanism used by DRM video players — the layer's contents become invisible to screen capture APIs.

### The SpaceWarp Node

The outermost wrapper is a `SpaceWarpNode` — this creates the "ripple" visual effect you see when a message is sent. The ripple expands outward from the send button, briefly warping the chat content. It's a subtle but characteristic Telegram animation.

## ChatControllerInteraction: The Callback Bag

Perhaps the most architecturally interesting piece is `ChatControllerInteraction`. This class is a massive collection of closures — **over 80 callback properties** — that message nodes use to communicate user actions back to the controller:

```swift
public final class ChatControllerInteraction: ChatControllerInteractionProtocol {
    public let openMessage: (Message, OpenMessageParams) -> Bool
    public let openPeer: (EnginePeer, ChatControllerInteractionNavigateToPeer,
                          MessageReference?, OpenPeerSource) -> Void
    public let openPeerMention: (String, Promise<Bool>?) -> Void
    public let openMessageContextMenu: (Message, Bool, ASDisplayNode,
                                        CGRect, UIGestureRecognizer?, CGPoint?) -> Void
    public let updateMessageReaction: (Message, ChatControllerInteractionReaction,
                                       Bool, ContextExtractedContentContainingView?) -> Void
    public let navigateToMessage: (MessageId, MessageId, NavigateToMessageParams) -> Void
    public let toggleMessagesSelection: ([MessageId], Bool) -> Void
    public let sendCurrentMessage: (Bool, ChatSendMessageEffect?) -> Void
    public let sendSticker: (FileMediaReference, Bool, Bool, String?,
                             Bool, UIView, CGRect, CALayer?, [ItemCollectionId]) -> Bool
    public let sendGif: (FileMediaReference, UIView, CGRect, Bool, Bool) -> Bool
    public let openUrl: (OpenUrl) -> Void
    public let setupReply: (MessageId) -> Void
    public let performTextSelectionAction: (Message?, Bool, NSAttributedString,
                                            TextSelectionAction) -> Void
    public let openWebView: (String, String, Bool, ChatOpenWebViewSource) -> Void
    public let seekToTimecode: (Message, Double, Bool) -> Void
    // ... 60+ more callbacks
```

Why closures instead of a delegate protocol? Because `ChatControllerInteraction` is shared across **hundreds** of message nodes. Each message bubble node, text node, media node, and action button receives a reference to this object. A protocol with 80+ methods would be unwieldy and would require a single conforming type. With closures, the ChatController can wire up each callback independently, and the compiler ensures every callback is provided at construction time.

The interaction object also carries mutable UI state that message nodes need to read:

```swift
public var canPlayMedia: Bool = false
public var hiddenMedia: [MessageId: [Media]] = [:]
public var expandedTranslationMessageStableIds: Set<UInt32> = Set()
public var selectionState: ChatInterfaceSelectionState?
public var highlightedState: ChatInterfaceHighlightedState?
public var pollActionState: ChatInterfacePollActionState = ChatInterfacePollActionState()
public var currentPollMessageWithTooltip: MessageId?
public var stickerSettings: ChatInterfaceStickerSettings
public var searchTextHighightState: (String, [MessageIndex])?
public var unreadMessageRange: [UnreadMessageRangeKey: Range<MessageId.Id>] = [:]
public var chatIsRotated: Bool
```

This is an explicit design choice: rather than having each message node subscribe to separate state signals (which would create thousands of subscriptions), the controller updates mutable properties on the shared interaction object and then tells the list view to re-render affected cells.

## ContentData: The Reactive Data Pipeline

Starting around line 131 of `ChatController+ReloadChatLocation.swift`, the `ContentData` class manages reactive subscriptions to the chat's data:

```swift
func reloadChatLocation(
    chatLocation: ChatLocation,
    chatLocationContextHolder: Atomic<ChatLocationContextHolder?>,
    historyNode: ChatHistoryListNodeImpl,
    apply: @escaping ((ContainedViewLayoutTransition?) -> Void) -> Void
) {
    self.contentDataReady.set(false)
    self.contentDataDisposable?.dispose()

    let contentData = ChatControllerImpl.ContentData(
        context: self.context,
        chatLocation: chatLocation,
        chatLocationContextHolder: chatLocationContextHolder,
        initialSubject: self.subject,
        mode: self.mode,
        configuration: configuration,
        adMessagesContext: self.chatDisplayNode.adMessagesContext,
        presentationData: self.presentationData,
        historyNode: historyNode,
        // ...
    )
    self.pendingContentData = (contentData, historyNode)

    self.contentDataDisposable = (contentData.isReady.get()
    |> filter { $0 }
    |> take(1)
    |> deliverOnMainQueue).startStrict(next: { [weak self, weak contentData] _ in
        guard let self, let contentData else { return }

        apply({ forceAnimationTransition in
            self.contentData = contentData
            self.pendingContentData = nil
            self.contentDataUpdated(synchronous: true,
                                    forceAnimationTransition: forceAnimationTransition,
                                    previousState: contentData.state)
            // ...
            self.contentDataReady.set(true)

            contentData.onUpdated = { [weak self] previousState in
                self?.contentDataUpdated(synchronous: false,
                                         forceAnimationTransition: nil,
                                         previousState: previousState)
            }
        })
    })
}
```

The `ContentData` object aggregates multiple signal subscriptions into a single reactive pipeline. When all the data is ready (peer info, cached data, pinned messages, etc.), it signals `isReady` and the controller applies the state. Subsequent updates flow through the `onUpdated` callback.

The `contentDataUpdated` method (line 228) is where all the pieces come together. It receives the previous state and the new state, computes what changed, decides whether to animate, and calls `updateChatPresentationInterfaceState`:

```swift
func contentDataUpdated(synchronous: Bool,
                        forceAnimationTransition: ContainedViewLayoutTransition?,
                        previousState: ContentData.State) {
    guard let contentData = self.contentData else { return }

    var animated = false
    if self.presentationInterfaceState.adMessage?.id != contentData.state.adMessage?.id {
        animated = true
    }
    if let peer = previousState.renderedPeer?.peer as? TelegramSecretChat,
       let updated = contentData.state.renderedPeer?.peer as? TelegramSecretChat,
       peer.embeddedState != updated.embeddedState {
        animated = true
    }
    if previousState.pinnedMessage != contentData.state.pinnedMessage {
        animated = true
    }
    // ... more change detection

    var transition: ContainedViewLayoutTransition =
        animated ? .animated(duration: 0.4, curve: .spring) : .immediate

    self.updateChatPresentationInterfaceState(transition: transition, interactive: false,
                                              force: true) { state in
        var state = state
        state = state.updatedPeer({ _ in contentData.state.renderedPeer })
        state = state.updatedContactStatus(contentData.state.contactStatus)
        state = state.updatedHasBots(contentData.state.hasBots)
        // ... apply all new values
        return state
    }
}
```

## Lifecycle: viewWillAppear and viewDidAppear

The controller's lifecycle methods reveal the complexity of coordinating a chat screen:

**viewWillAppear** (line 6803):
- Refreshes poll actions for visible messages
- Refocuses on unread messages in reply threads
- Synchronizes message counters with the server
- Activates scheduled text input if pending
- Builds the chat navigation stack for the back button long-press menu

**viewDidAppear** (line 6900):
- Disables experimental snap-scroll
- Enables read-history tracking (marking messages as read)
- Enables content animations (stickers, GIFs)
- Loads the input panels (emoji keyboard, sticker picker)
- Sets up the recently-used inline bots list
- Initializes **Raise to Listen** — the feature where raising the phone to your ear auto-plays the last voice message, or starts recording a new one:

```swift
self.raiseToListen = RaiseToListenManager(shouldActivate: { [weak self] in
    guard let strongSelf = self else { return false }
    if !strongSelf.context.sharedContext.currentMediaInputSettings
        .with({ $0.enableRaiseToSpeak }) {
        return false
    }
    if strongSelf.effectiveNavigationController?.topViewController !== strongSelf {
        return false
    }
    if strongSelf.presentationInterfaceState.inputTextPanelState
        .mediaRecordingState != nil {
        return false
    }
    if strongSelf.firstLoadedMessageToListen() != nil
        || strongSelf.chatDisplayNode.isTextInputPanelActive {
        return true
    }
    return false
}, activate: { [weak self] in
    self?.activateRaiseGesture()
}, deactivate: { [weak self] in
    self?.deactivateRaiseGesture()
})
```

Raise to Listen uses the proximity sensor to detect when the phone is near the user's ear. The `shouldActivate` closure runs a gauntlet of checks — is the feature enabled? Is this the top controller? Is the user not already recording? Is there a voice message to play? — before allowing activation.

## Input Panel Management

The chat input area is one of the most complex parts of the UI. It's not a single component — it's a stack of interchangeable panels:

### The Text Input Panel

`ChatTextInputPanelNode` is the default input panel with the text field, attachment button, and send button. It handles:

- Multi-line text editing with auto-grow
- @mention and /command autocomplete triggers
- Emoji and sticker suggestions
- Voice/video message recording (tap and hold the microphone button)
- Slow-mode countdown overlay
- Send-as-peer selection

### Accessory Panels

Above the input panel, accessory panels appear for contextual actions:

- **ReplyAccessoryPanelNode** — shows the message being replied to
- **ForwardAccessoryPanelNode** — shows forwarded message info
- **EditAccessoryPanelNode** — shows the message being edited
- **SuggestPostAccessoryPanelNode** — shows suggested post info
- **WebpagePreviewAccessoryPanelNode** — shows URL preview being composed

### The Full Keyboard

When the user taps the emoji button, the text input panel is replaced by a full-screen `ChatInputNode` containing `ChatEntityKeyboardInputNode` — a tabbed keyboard with emoji, stickers, and GIFs. This is the component that manages the "smooth" keyboard transition where the custom keyboard slides in from below, matching the system keyboard's position exactly.

### Input Context Panels

As the user types, context panels appear above the input:

```swift
var contextQueryStates: [ChatPresentationInputQueryKind:
    (ChatPresentationInputQuery, Disposable)] = [:]
```

These handle:
- `@` — mention suggestions
- `/` — bot command suggestions
- `#` — hashtag suggestions
- Inline bot queries (`@gif dog`)
- Emoji suggestions from text

Each context query creates its own signal subscription, and results appear in an `ChatInputContextPanelNode` that slides up from the input bar.

## The chatDisplayNode Accessor

One detail worth noting — `ChatControllerImpl` accesses its display node through a computed property that force-casts:

```swift
var chatDisplayNode: ChatControllerNode {
    get {
        return super.displayNode as! ChatControllerNode
    }
}
```

This is the standard pattern throughout Telegram — `ViewController` stores the display node as `ASDisplayNode`, and each subclass provides a typed accessor. The force-cast is safe because the controller creates the node in `loadDisplayNode()` and always creates the correct type.

## Navigation Integration

The chat controller integrates deeply with Telegram's custom navigation system:

### Title View

The navigation bar's title view is a custom `ChatNavigationBarTitleView` (stored as `chatTitleView`). It shows:
- The peer's name and online status
- Typing indicators ("John is typing...")
- Recording indicators ("John is recording audio...")
- Encrypted chat lock icon
- Group call status

### Navigation Buttons

The controller manages four navigation button positions:

```swift
var leftNavigationButton: ChatNavigationButton?
var rightNavigationButton: ChatNavigationButton?
var secondaryRightNavigationButton: ChatNavigationButton?
var chatInfoNavigationButton: ChatNavigationButton?
```

The right side shows the peer's avatar as a navigation button (via `ChatAvatarNavigationNode`), and the left side can show a close button, edit button, or selection-mode actions depending on the current state.

### Back Button Long-Press

The chat controller provides a custom back button experience. When the user long-presses the back arrow, a context menu appears showing the navigation stack of recent chats, letting the user jump back multiple steps:

```swift
if !chatNavigationStack.isEmpty,
   let backButtonNode = self.chatDisplayNode.navigationBar?.backButtonNode
        as? ContextControllerSourceNode {
    backButtonNode.isGestureEnabled = true
    backButtonNode.activated = { [weak self] gesture, _ in
        PeerInfoScreenImpl.displayChatNavigationMenu(
            context: strongSelf.context,
            chatNavigationStack: chatNavigationStack,
            nextFolderId: nextFolderId,
            parentController: strongSelf,
            backButtonView: backButtonNode.view,
            navigationController: navigationController,
            gesture: gesture
        )
    }
}
```

## Audio and Video Recording

The controller manages both audio and video recording through a promise-based system:

```swift
var audioRecorderValue: ManagedAudioRecorder?
var audioRecorder = Promise<ManagedAudioRecorder?>()
var audioRecorderDisposable: Disposable?

var videoRecorderValue: VideoMessageCameraScreen?
var videoRecorder = Promise<VideoMessageCameraScreen?>()
var videoRecorderDisposable: Disposable?
```

When the user holds the microphone button, the controller sets `audioRecorder.set(.single(recorder))`, which triggers the `audioRecorderDisposable` subscription to show the recording UI. If the user swipes up, the recording switches to locked mode. If they swipe left, it cancels. The same pattern applies to video messages, where `VideoMessageCameraScreen` provides the circular video recording overlay.

Recording state flows through `ChatRecordingActivity`:

```swift
enum ChatRecordingActivity {
    case voice
    case instantVideo
    case none
}
```

This state is sent to the server as a typing indicator, so other participants see "recording voice message..." or "recording video message...".

## Performance Optimizations

Several key optimizations keep the chat screen responsive:

### Lazy Node Loading

The display node is loaded lazily via `loadDisplayNode()`:

```swift
override public func loadDisplayNode() {
    self.loadDisplayNodeImpl()
    self.galleryPresentationContext.view = self.view
}
```

This means the entire node hierarchy — wallpaper, message list, input panels — is only created when the view is actually needed, not when the controller is initialized.

### Preloading Next Chat

The controller preloads the next chat's data when the user is near the end of a chat list:

```swift
var preloadNextChatPeerId: EnginePeer.Id? = nil
let preloadNextChatPeerIdDisposable = MetaDisposable()
```

### Wallpaper Ready Gate

The chat doesn't show content until the wallpaper is rendered:

```swift
let wallpaperReady = Promise<Bool>()
let presentationReady = Promise<Bool>()
```

These promises gate the controller's `ready` signal, preventing a flash of white background before the wallpaper appears.

### Message Transition Coordination

Send/receive animations are managed by `ChatMessageTransitionNodeImpl`. This node orchestrates the animation of a message bubble from the input area to its final position in the list, or from a forwarded message to its new location. It lives outside the history node's scroll container so it can animate across the entire screen.

## How Everything Connects

To understand the full data flow, trace what happens when a new message arrives:

1. **WebSocket update** → `AccountStateManager` processes `updateNewMessage`
2. **Postbox transaction** → message is stored, `MessageHistoryView` updates
3. **ChatHistoryListNodeImpl** — observes the history view, inserts a new `ListViewItem`
4. **ChatMessageItemImpl** — wraps the `Message` into a list item with headers and merge info
5. **ChatMessageBubbleItemNode** — async-layouts the bubble with text, media, and status
6. **ChatControllerNode** — the list view calls back with visibility changes
7. **ChatControllerImpl** — updates unread count, triggers read receipts, plays notification sound

The entire pipeline runs in under 16ms for a text message on modern hardware, maintaining 60fps scrolling even while new messages arrive.

## Summary

`ChatControllerImpl` is a masterclass in managing complexity. Its key architectural decisions:

- **Single immutable state** (`ChatPresentationInterfaceState`) that all UI derives from
- **Closure-based interaction** (`ChatControllerInteraction`) instead of delegates, shared across hundreds of nodes
- **Reactive data pipeline** (`ContentData`) that aggregates dozens of signal subscriptions into coordinated state updates
- **Layered node hierarchy** (`ChatControllerNode`) with pluggable input panels, accessory panels, and overlay containers
- **Promise-gated readiness** to prevent visual artifacts during load

The result is a chat screen that feels instantaneous and seamless despite orchestrating dozens of concurrent data streams, hundreds of message nodes, and features spanning nearly every module in a 274-module codebase. It's the most ambitious view controller I've seen in any iOS app, and understanding how it works illuminates the architectural patterns that make the rest of Telegram possible.
