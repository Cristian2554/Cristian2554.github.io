---
layout: post
title:  "Creating Game Phases with State Trees"
date:   2025-09-27 
categories: jekyll update
---

Recently, a colleague and I started to prototype a new game which combines the combat structure of Pokémon, the collection aspects of some card games, and dice mechanics to determine who would win each combat (I’m not a designer, but I’ll be happy to go into more details in another post if its interesting). As with any other prototype, the idea is to be quick and scalable, but this one also has a personal goal to try new tech that I have not used before, so I decided to use StateTree as the main driver to the game flow. 

As per Epic, “StateTree is a general-propuse hierarchical state machine that combines the Selectors from behavior trees with States and Transitions from state machines”. I’ve seen people all around the interwebs use them for all sort of things, from replacing the state of an actor we would normally represent with an enum to making full AI systems that replace Behavior Trees. I struggled in the past while making a Game Mode that has different stages that grow infinitely, are not reusable and is easy to break whenever your designer wants to add an small timer before a single phase of the game. StateTree seemed like a good solution for this, so I gave it a try. 

## The Goal

Similar to Pokémon combat, our game has stages in which the next player to have a turn is selected, it draws a die from its Dice Bag, throws it and then some effects will happen once the die lands. This will go on until the dices in the player’s “hand” are over, making the round end and applying the effects that the face up of the dice in the field states. Finally all players draw dice again and the cycle continues until someone ends. This is as much detail as I think we need to explain the technical part, since this is subject to change soon. 

## The implementation

 At its simplest form, StateTree works by having an Actor with an StateTreeComponent run a StateTree asset (Which is an object derived from DataAsset). So I started by adding a StateTreeComponent to my Combat Game Mode and assigning it the ST_CombatGameFlow StateTree to it. I imagine that in the future the this Data Asset could be swapped whenever is needed, maybe to prototype new game flows or entirely new game modes. 

![image.png](/assets/images/statetree-showcase.png){: .center-image }

An StateTree DataAsset is basically a collection of states and substates that can go to other states whenever a condition is met. I find that StateTrees seem to be a hybrid between a Behavior Tree and a traditional finite state machine. It has functionality to both pick child nodes based on weight or also behave only by transition conditions like a regular state machine. I assume is intended so you can give it any use you want. The key part about them is the transitions, which can be selected in a more smart way by adding weights and more complex rules. For my use case, I wanted it to be more linear, so I added transitions to either their next state or a particular state whenever I needed them. 

Each time we enter a state I want something to happen, which is where State Tree Tasks come into place. Tasks are pieces of code that will run when you state enters, exits or ticks, and will define if the state was completed, either successfully or not. My main goal was to have modularity in mind, so I created different tasks for small actions. Some example of this are:

- Pick the Current Active Agent (An agent a term I used to refer to either a Player Controller or an AI Controller, as the game could be played against an AI or another player)
- Ask the Current Active Agent to draw a number of dice from its bag into his “hand”.
- Ask the Current Active Agent to pick a die out of its hand. If its a player to do it through an UI interface, and if its an AI to do it based on its logic.
- Ask the Current Active Agent to throw its picked die to the battle field.
- Check the end of round behaviors of all dice on the field and execute them.

![image.png](/assets/images/statetree-task-lists.png){: .center-image }

## Making State Tree Tasks

Before using Tasks I assumed they would be classes, but they happen to be child structs from FStateTreeTaskCommonBase. They can override some functions, among them the *EnterState*, *ExitState*, and *Tick*.

```cpp
virtual EStateTreeRunStatus EnterState(FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const override;

virtual EStateTreeRunStatus Tick(FStateTreeExecutionContext& Context, const float DeltaTime) const override;

virtual void ExitState(FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const override;
```

The Context parameter provides access to things outside the state, like the Actor owner of the component running the StateTree, or even the StateTree itself. Tasks are meant to live only while the state is alive, so they can’t access the variables of the struct itself. To maintain data during the life of the Task we can use Instance Data, which is an struct of that that can be exposed to blueprints and that retains information while the Task runs. 

An State Tree Task with parameters exposed to blueprint will look something like this:

```cpp
enum class EStateTreeRunStatus : uint8;
struct FStateTreeTransitionResult;

UENUM(BlueprintType)
enum class EPopulateType : uint8
{
    PopulateActiveAgent UMETA(DisplayName = "Populate Current Active Agent"),
    PopulateAllAgents UMETA(DisplayName = "Populate All Agents")
};

USTRUCT()
struct DICEBATTLE_API FSTPopulateDieHandInstanceData
{
    GENERATED_BODY()
    
    UPROPERTY(EditAnywhere, Category = Parameter)
    int32 DicePerHand = 3;
    
    UPROPERTY(EditAnywhere, Category = Parameter)
    EPopulateType PopulateType = EPopulateType::PopulateActiveAgent;

};

USTRUCT(meta = (DisplayName = "Request the Active Agent to Draw his Hand", Category = "Game Flow"))
struct DICEBATTLE_API FSTPopulateDieHand : public FStateTreeTaskCommonBase
{
    GENERATED_BODY()

    using FInstanceDataType = FSTPopulateDieHandInstanceData;

    FSTPopulateDieHand() = default;

    virtual const UStruct* GetInstanceDataType() const override { return FInstanceDataType::StaticStruct(); }

    virtual EStateTreeRunStatus EnterState (FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const override;
};
```

![image.png](/assets/images/statetree-showcase-selected.png){: .center-image }