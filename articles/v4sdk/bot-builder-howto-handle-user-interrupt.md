---
title: Handle user interruptions | Microsoft Docs
description: Learn how to handle user interrupt and direct conversation flow.
keywords: interrupt, interruptions, switching topic, break
author: v-ducvo
ms.author: v-ducvo
manager: kamrani
ms.topic: article
ms.prod: bot-framework
ms.date: 04/17/2018
ms.reviewer:
monikerRange: 'azure-bot-service-4.0'
---

# Handle user interruptions

[!INCLUDE [pre-release-label](../includes/pre-release-label.md)]

Handling interruptions is an important aspect of a robust bot.

While you may think that your users will follow your defined conversation flow step by step, chances are good that they will change their minds or ask a question in the middle of the process instead of answering the question. In these situations, how would your bot handle the user's input? What would the user experience be like? How would you maintain user state data? Handling interruptions means making sure your bot is prepared to handle situations like this.

There is no right answer to these questions as each situation is unique to the scenario your bot is designed to handle. In this topic, we will explore some common ways to handle user interruptions and suggest some ways to implement them in your bot.

## Handle expected interruptions

A procedural conversation flow has a core set of steps that you want to lead the user through, and any user actions that vary from those steps are potential interruptions. In a normal flow, there are interruptions that you can anticipate.

**Table reservation**
In a table reservation bot, the core steps might be to ask the user for a date and time, the size of the party, and the reservation name. In that process, some expected interruptions you could anticipate may include:

* `cancel`: To exit the process.
* `help`: To provide additional guidance about this process.
* `more info`: To give hints and suggestions or provide alternative ways of reserving a table (e.g.: an email address or phone number to contact).
* `show list of available tables`: If that is an option; show a list of tables available for the date and time the user wanted.

**Order dinner**
In an order dinner bot, the core steps might be to provide a list of menu items and allow the user to add items to their cart. In this process, some expected interruptions you could anticipate may include:

* `cancel`: To exit the ordering process.
* `more info`: To provide dietary detail about each menu item.
* `help`: To provide help on how to use the system.
* `process order`: To process the order.

You could provide these to the user as a list of **suggested actions** or as a hint so the user is at least aware of what commands they can send that the bot would understand.

For example, in the order dinner flow, you can provide expected interruptions along with the menu items. In this case, the menu items are sent as an array of `choices`.

# [C#](#tab/csharptab)

We'll define the dialog set as a subclass of **DialogSet**.

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Bot.Builder;
using Microsoft.Bot.Builder.Dialogs;
using Microsoft.Bot.Builder.Dialogs.Choices;

public class OrderDinnerDialogs : DialogSet
{
    public OrderDinnerDialogs(IStatePropertyAccessor<DialogState> dialogStateAccessor)
        : base(dialogStateAccessor)
    {
    }
}
```

We'll define a couple inner classes to describe the menu.

```cs
/// <summary>
/// Contains information about an item on the menu.
/// </summary>
public class DinnerItem
{
    public string Description { get; set; }

    public double Price { get; set; }
}

/// <summary>
/// Describes the dinner menu, including the items on the menu and options for
/// interrupting the ordering process.
/// </summary>
public class DinnerMenu
{
    /// <summary>Gets the items on the menu.</summary>
    public static Dictionary<string, DinnerItem> MenuItems { get; } = new Dictionary<string, DinnerItem>
    {
        ["Potato salad"] = new DinnerItem { Description = "Potato Salad", Price = 5.99 },
        ["Tuna sandwich"] = new DinnerItem { Description = "Tuna Sandwich", Price = 6.89 },
        ["Clam chowder"] = new DinnerItem { Description = "Clam Chowder", Price = 4.50 },
    };

    /// <summary>Gets all the "interruptions" the bot knows how to process.</summary>
    public static List<string> Interrupts { get; } = new List<string>
    {
        "More info", "Process order", "Help", "Cancel",
    };

    /// <summary>Gets all of the valid inputs a user can make.</summary>
    public static List<string> Choices { get; }
        = MenuItems.Select(c => c.Key).Concat(Interrupts).ToList();
}
```

# [JavaScript](#tab/jstab)

```javascript
var dinnerMenu = {
    choices: ["Potato Salad - $5.99", "Tuna Sandwich - $6.89", "Clam Chowder - $4.50",
            "more info", "Process order", "Cancel"],
    "Potato Salad - $5.99": {
        Description: "Potato Salad",
        Price: 5.99
    },
    "Tuna Sandwich - $6.89": {
        Description: "Tuna Sandwich",
        Price: 6.89
    },
    "Clam Chowder - $4.50": {
        Description: "Clam Chowder",
        Price: 4.50
    }
}
```

---

In your ordering logic, you can check for them using string matching or regular expressions.

# [C#](#tab/csharptab)

First, we need to define a helper to keep track of our orders.

```cs
/// <summary>Helper class for storing the order.</summary>
public class Order
{
    public double Total { get; set; } = 0.0;

    public List<DinnerItem> Items { get; set; } = new List<DinnerItem>();

    public bool ReadyToProcess { get; set; } = false;

    public bool OrderProcessed { get; set; } = false;
}
```

Add some constants to track the IDs we'll need.

```csharp
/// <summary>The ID of the top-level dialog.</summary>
public const string MainDialogId = "mainMenu";

/// <summary>The ID of the choice prompt.</summary>
public const string ChoicePromptId = "choicePrompt";

/// <summary>The ID of the order card value, tracked inside the dialog.</summary>
public const string OrderCartId = "orderCart";
```

Update the constructor to add a choice prompt and our waterfall dialog. We'll also define the methods that implement the waterfall steps.

```cs
public OrderDinnerDialogs(IStatePropertyAccessor<DialogState> dialogStateAccessor)
    : base(dialogStateAccessor)
{
    // Add a choice prompt for the dialog.
    Add(new ChoicePrompt(ChoicePromptId));

    // Define and add the main waterfall dialog.
    WaterfallStep[] steps = new WaterfallStep[]
    {
        PromptUserAsync,
        ProcessInputAsync,
    };

    Add(new WaterfallDialog(MainDialogId, steps));
}

/// <summary>
/// Defines the first step of the main dialog, which is to ask for input from the user.
/// </summary>
/// <param name="stepContext">The current waterfall step context.</param>
/// <param name="cancellationToken">The cancellation token.</param>
/// <returns>The task to perform.</returns>
private async Task<DialogTurnResult> PromptUserAsync(WaterfallStepContext stepContext, CancellationToken cancellationToken)
{
    // Initialize order, continuing any order that was passed in.
    Order order = (stepContext.Options is Order oldCart && oldCart != null)
        ? new Order
        {
            Items = new List<DinnerItem>(oldCart.Items),
            Total = oldCart.Total,
            ReadyToProcess = oldCart.ReadyToProcess,
            OrderProcessed = oldCart.OrderProcessed,
        }
        : new Order();

    // Set the order cart in dialog state.
    stepContext.Values[OrderCartId] = order;

    // Prompt the user.
    return await stepContext.PromptAsync(
        "choicePrompt",
        new PromptOptions
        {
            Prompt = MessageFactory.Text("What would you like for dinner?"),
            RetryPrompt = MessageFactory.Text(
                "I'm sorry, I didn't understand that. What would you like for dinner?"),
            Choices = ChoiceFactory.ToChoices(DinnerMenu.Choices),
        },
        cancellationToken);
}

/// <summary>
/// Defines the second step of the main dialog, which is to process the user's input, and
/// repeat or exit as appropriate.
/// </summary>
/// <param name="stepContext">The current waterfall step context.</param>
/// <param name="cancellationToken">The cancellation token.</param>
/// <returns>The task to perform.</returns>
private async Task<DialogTurnResult> ProcessInputAsync(WaterfallStepContext stepContext, CancellationToken cancellationToken)
{
    // Get the order cart from dialog state.
    Order order = stepContext.Values[OrderCartId] as Order;

    // Get the user's choice from the previous prompt.
    string response = (stepContext.Result as FoundChoice).Value;

    if (response.Equals("process order", StringComparison.InvariantCultureIgnoreCase))
    {
        order.ReadyToProcess = true;

        await stepContext.Context.SendActivityAsync(
            "Your order is on it's way!",
            cancellationToken: cancellationToken);

        // In production, you may want to store something more helpful.
        // "Process" the order and exit.
        order.OrderProcessed = true;
        return await stepContext.EndDialogAsync(null, cancellationToken);
    }
    else if (response.Equals("cancel", StringComparison.InvariantCultureIgnoreCase))
    {
        // Cancel the order.
        await stepContext.Context.SendActivityAsync(
            "Your order has been canceled",
            cancellationToken: cancellationToken);

        // Exit without processing the order.
        return await stepContext.EndDialogAsync(null, cancellationToken);
    }
    else if (response.Equals("more info", StringComparison.InvariantCultureIgnoreCase))
    {
        // Send more information about the options.
        string message = "More info: <br/>" +
            "Potato Salad: contains 330 calories per serving. Cost: 5.99 <br/>"
            + "Tuna Sandwich: contains 700 calories per serving. Cost: 6.89 <br/>"
            + "Clam Chowder: contains 650 calories per serving. Cost: 4.50";
        await stepContext.Context.SendActivityAsync(
            message,
            cancellationToken: cancellationToken);

        // Continue the ordering process, passing in the current order cart.
        return await stepContext.ReplaceDialogAsync(MainDialogId, order, cancellationToken);
    }
    else if (response.Equals("help", StringComparison.InvariantCultureIgnoreCase))
    {
        // Provide help information.
        string message = "To make an order, add as many items to your cart as " +
            "you like. Choose the `Process order` to check out. " +
            "Choose `Cancel` to cancel your order and exit.";
        await stepContext.Context.SendActivityAsync(
            message,
            cancellationToken: cancellationToken);

        // Continue the ordering process, passing in the current order cart.
        return await stepContext.ReplaceDialogAsync(MainDialogId, order, cancellationToken);
    }

    // We've checked for expected interruptions. Check for a valid item choice.
    if (!DinnerMenu.MenuItems.ContainsKey(response))
    {
        await stepContext.Context.SendActivityAsync("Sorry, that is not a valid item. " +
            "Please pick one from the menu.");

        // Continue the ordering process, passing in the current order cart.
        return await stepContext.ReplaceDialogAsync(MainDialogId, order, cancellationToken);
    }
    else
    {
        // Add the item to cart.
        DinnerItem item = DinnerMenu.MenuItems[response];
        order.Items.Add(item);
        order.Total += item.Price;

        // Acknowledge the input.
        await stepContext.Context.SendActivityAsync(
            $"Added `{response}` to your order; your total is ${order.Total:0.00}.",
            cancellationToken: cancellationToken);

        // Continue the ordering process, passing in the current order cart.
        return await stepContext.ReplaceDialogAsync(MainDialogId, order, cancellationToken);
    }
}
```

# [JavaScript](#tab/jstab)

Notice that the code checks for and handles interruptions _first_, then proceeds to the next logical step.

<!--@Ben: where did `orderCart` come from? was it missing all along? I've changed the C# Order class to contain a list of items.-->

```javascript
// Helper dialog to repeatedly prompt user for orders
dialogs.add('orderPrompt', [
    async function(step, orderCart) {
        // Define a new cart of one does not exists
        if (!orderCart) {
            // Initialize a new cart
            step.values.orderCart = {
                orders: [],
                total: 0
            };
        } else {
            step.values.orderCart = orderCart;
        }
        return await step.prompt('choicePrompt', "What would you like?", dinnerMenu.choices);
    },
    async function(step) {
        const choice = step.result;
        if (choice.value.match(/process order/ig)) {
            if (step.values.orderCart.orders.length > 0) {
                // Process the order by returning the order to the parent dialog
                return await step.endDialog(step.values.orderCart);
            } else {
                await step.context.sendActivity("Your cart was empty. Please add at least one item to the cart.");
                // Ask again
                return await step.replaceDialog('orderPrompt');
            }
        } else if (choice.value.match(/cancel/ig)) {
            step.values.orderCart = {
                orders: [],
                total: 0
            };
            await step.context.sendActivity("Your order has been canceled.");
            await step.endDialog(choice.value);
        } else if (choice.value.match(/more info/ig)) {
            var msg = "More info: <br/>Potato Salad: contains 330 calories per serving. <br/>"
                + "Tuna Sandwich: contains 700 calories per serving. <br/>"
                + "Clam Chowder: contains 650 calories per serving."
            await step.context.sendActivity(msg);

            // Ask again
            return await step.replaceDialog('orderPrompt');
        } else if (choice.value.match(/help/ig)) {
            var msg = `Help: <br/>To make an order, add as many items to your cart as you like then choose the "Process order" option to check out.`
            await step.context.sendActivity(msg);

            // Ask again
            return await step.replaceDialog('orderPrompt');
        } else {
            var choice = dinnerMenu[choice.value];

            // Only proceed if user chooses an item from the menu
            if (!choice) {
                await step.context.sendActivity("Sorry, that is not a valid item. Please pick one from the menu.");

                // Ask again
                await step.replaceDialog('orderPrompt');
            } else {
                // Add the item to cart
                step.values.orderCart.orders.push(choice);
                step.values.orderCart.total += dinnerMenu[choice.value].Price;

                await step.context.sendActivity(`Added to cart: ${choice.value}. <br/>Current total: $${ step.values.orderCart.total}`);

                // Ask again
                return await step.replaceDialog('orderPrompt');
            }
        }
    }
]);
```

---

## Handle unexpected interruptions

There are interruptions that are out of scope of what your bot is designed to do.
While you cannot anticipate all interruptions, there are patterns of interruptions that you can program your bot to handle.

### Switching topic of conversations

What if the user is in the middle of one conversation and wants to switch to another conversation? For example, your bot can reserve a table and order dinner.
While the user is in the _reserve a table_ flow, instead of answering the question for "How many people are in your party?", the user sends the message "order dinner". In this case, the user changed their mind and wants to engage in a dinner ordering conversation instead. How should you handle this interruption?

You can switch topics to the order dinner flow or you can make it a sticky issue by telling the user that you are expecting a number and reprompt them. If you do allow them to switch topics, you then have to decide if you will save the progress so that the user can pick up from where they left off or you could delete all the information you have collected so that they will have to start that process all over next time they want to reserve a table. For more information about managing user state data, see [Save state using conversation and user properties](bot-builder-howto-v4-state.md).

### Apply artificial intelligence

For interruptions that are not in scope, you can try to guess what the user intent is. You can do this using AI services such as QnA Maker, LUIS, or your custom logic, then offer up suggestions for what the bot thinks the user wants.

For example, while in the middle of the reserve table flow, the user says, "I want to order a burger". This is not something the bot knows how to handle from this conversation flow. Since the current flow has nothing to do with ordering, and the bot's other conversation command is "order dinner", the bot does not know what to do with this input. If you apply LUIS, for example, you could train the model to recognize that they want to order food (e.g.: LUIS can return an "orderFood" intent). Thus, the bot could respond with, "It seems you want to order food. Would you like to switch to our order dinner process instead?" For more information on training LUIS and detecting user intents, see [Use LUIS for language understanding](bot-builder-howto-v4-luis.md).

### Default response

If all else fails, you should send a default response instead of doing nothing and leaving the user wondering what is going on. The default response should tell the user what commands the bot understands so the user can get back on track.

You can check against the context **responded** flag at the end of the bot logic to see if the bot sent anything back to the user during the turn. If the bot processes the user's input but does not respond, chances are that the bot does not know what to do with the input. In that case, you can catch it and send the user a default message.

The default for this bot is to give the user the `mainMenu` dialog. It's up to you to decide what experience your user will have in this situation for your bot.

# [C#](#tab/csharptab)

```cs
// Check whether we replied. If not then clear the dialog stack and present the main menu.
if (!turnContext.Responded)
{
    await dc.CancelAllDialogsAsync(cancellationToken);
    await dc.BeginDialogAsync(OrderDinnerDialogs.MainDialogId, null, cancellationToken);
}
```

# [JavaScript](#tab/jstab)

```javascript
// Check to see if anyone replied. If not then clear all the stack and present the main menu
if (!context.responded) {
    await dc.cancelAllDialogs();
    return await step.beginDialog('mainMenu');
}
```

---

## Handling Global Interruptions

In the above example, we are handling interruptions that might occur at a specific turn in a specific dialog. What if we want to handle global interruptions - the kind that might happen at any time?

This can be achieved by putting our interruption handling logic in the bot's main handler - the function that processes the incoming `turnContext` and decides what to do with it.

In the example below, the _first thing_ the bot does is check incoming message text for a sign that the user needs help or wants to cancel - two very common interruptions for bots to encounter. After this check is complete, the bot calls `dc.continueDialog()` to process any active dialogs still pending.

# [C#](#tab/csharptab)

```cs
// Check for top-level interruptions.
string utterance = turnContext.Activity.Text.Trim().ToLowerInvariant();

if (utterance == "help")
{
    // Start a general help dialog. Dialogs already on the stack remain and will continue
    // normally if the help dialog exits normally.
    await dc.BeginDialogAsync(OrderDinnerDialogs.HelpDialogId, null, cancellationToken);
}
else if (utterance == "cancel")
{
    // Cancel any dialog on the stack.
    await turnContext.SendActivityAsync("Canceled.", cancellationToken: cancellationToken);
    await dc.CancelAllDialogsAsync(cancellationToken);
}
else
{
    await dc.ContinueDialogAsync(cancellationToken);

    // Check whether we replied. If not then clear the dialog stack and present the main menu.
    if (!turnContext.Responded)
    {
        await dc.CancelAllDialogsAsync(cancellationToken);
        await dc.BeginDialogAsync(OrderDinnerDialogs.MainDialogId, null, cancellationToken);
    }
}
```

# [JavaScript](#tab/jstab)

```javascript
// Listen for incoming activity 
server.post('/api/messages', (req, res) => {
    // Route received activity to adapter for processing
    adapter.processActivity(req, res, async (context) => {
        const isMessage = context.activity.type === ActivityTypes.Message;
        // Create a DialogContext object.
        const dc = dialogs.createContext(context);

        if (isMessage) {

            const utterance = (context.activity.text || '').trim().toLowerCase();
            
            // Let's look for some interruptions first!
            if (utterance === 'help') {
                // Launch a new help dialog if the user asked for help
                await dc.beginDialog('helpDialog');
            } else if (utterance === 'cancel') {
                // Cancel any active dialogs if the user says cancel
                await dc.context.sendActivity('Canceled.');
                await dc.cancelAllDialogs();        
            }
            
            // If the bot hasn't yet responded...
            if (!context.responded) {
                // Continue any active dialog, which might send a response...
                await dc.continueDialog();

                // Finally, if the bot still hasn't sent a response, send instructions.
                if (!context.responded && isMessage) {
                    await dc.context.sendActivity(`Hi! I'm a sample bot!`);
                }
            }
        } else {
            await context.sendActivity(`[${context.activity.type} event detected]`);
        }
    });
});

```

---
