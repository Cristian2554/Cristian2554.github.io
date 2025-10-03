---
layout: post
title:  "Creating Game Phases with Unreal’s StateTree’s: Notes from a Prototype"
date:   2025-09-27 
categories: jekyll update
---

Recently, a colleague and I started to prototype a new game which combines the combat structure of Pokémon, the collection aspects of some card games, and dice mechanics to determine who would win each combat (I’m not a designer, but I’ll be happy to go into more details in another post if it’s interesting). As with any other prototype, the idea is to be quick and scalable, but this one also has a personal goal to try new tech that I have not used before, so I decided to use StateTree as the main driver to the game flow. 

As per Epic, “StateTree is a general-purpose hierarchical state machine that combines the Selectors from behavior trees with States and Transitions from state machines”. I’ve seen people all around the interwebs use them for all sorts of things, from replacing the state of an actor we would normally represent with an enum to making full AI systems that replace Behavior Trees. I struggled in the past while making a Game Mode that has different stages that grow infinitely, are not reusable and are easy to break whenever your designer wants to add a small timer before a single phase of the game. StateTree seemed like a good solution for this, so I gave it a try. 

So, If you are an Unreal Engine dev curious about practical StateTree uses, this post is for you. I started this prototype using Unreal Engine 5.5 and moved to 5.6 during the development. 

## TLDR

- I used StateTree’s from Unreal to drive my Game Flow States by placing it in my Game Mode.
- I created tasks to offload behaviors from the Game Mode into StateTree Tasks
- I found that it seems to be useful for games that have complex Game Flows, Like JRPG’s or games alike.

## The Goal

Similar to Pokémon combat, our game has stages in which the next player to have a turn is selected, it draws a die from their Dice Bag, throws it and then some effects will happen once the die lands. This will go on until the dice in the player’s “hand” are over, making the round end and applying the effects that the face up of the dice in the field states. Finally all players draw dice again and the cycle continues until someone ends. This is as much detail as I think we need to explain the technical part, since this is subject to change soon. 

## The implementation

At its simplest, StateTree works by having an Actor with a StateTreeComponent run a StateTree asset (Which is an object derived from DataAsset). So I started by adding a StateTreeComponent to my Combat Game Mode and assigning it the ST_CombatGameFlow StateTree to it. I imagine that in the future the this Data Asset could be swapped whenever is needed, maybe to prototype new game flows or entirely new game modes. 

![statetree-showcase.png](/assets/images/statetree-showcase.png){: .center-image }

An StateTree DataAsset is basically a collection of states and substates that can go to other states whenever a condition is met. I find that StateTrees seem to be a hybrid between a Behavior Tree and a traditional finite state machine. It has functionality to both pick child nodes based on weight or also behave only by transition conditions like a regular state machine. I assume is intended so you can give it any use you want. The key part about them is the transitions, which can be selected more intelligently by adding weights and more complex rules. For my use case, I wanted it to be more linear, so I added transitions to either their next state or a particular state whenever I needed them. 

Each time we enter a state I want something to happen, which is where StateTree’s Tasks come into play. Tasks are pieces of code that will run when you state enters, exits or ticks, and will define if the state was completed, either successfully or not. My main goal was to have modularity in mind, so I created different tasks for small actions. Some examples are:

- Pick the Current Active Agent (An agent a term I used to refer to either a Player Controller or an AI Controller, as the game could be played against an AI or another player)
- Ask the Current Active Agent to draw a number of dice from its bag into his “hand”.
- Ask the Current Active Agent to pick a die out of its hand. If it’s a player to do it through an UI interface, and if it’s an AI to do it based on its logic.
- Ask the Current Active Agent to throw its picked die to the battlefield.
- Check the end of round behaviors of all dice on the field and execute them.

![statetree-task-lists.png](/assets/images/statetree-task-lists.png){: .center-image }

## Making StateTree Tasks

Before using Tasks I assumed they would be classes, but they happen to be child structs from FStateTreeTaskCommonBase. They can override some functions, among them the *EnterState*, *ExitState*, and *Tick*.

```cpp
virtual EStateTreeRunStatus EnterState(FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const override;

virtual EStateTreeRunStatus Tick(FStateTreeExecutionContext& Context, const float DeltaTime) const override;

virtual void ExitState(FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const override;
```

The Context parameter provides access to things outside the state, like the Actor owner of the component running the StateTree, or even the StateTree itself. Tasks are meant to live only while the state is alive, so they can’t access the variables of the struct itself. To maintain data during the life of the Task we can use Instance Data, which is an struct of that can be exposed to blueprints and that retains information while the Task runs. 

A StateTree Task with parameters exposed to blueprint will look something like this:

```cpp
enum class EStateTreeRunStatus : uint8;
struct FStateTreeTransitionResult;

UENUM(BlueprintType)
enum class EPopulateType : uint8
{
	PopulateActiveAgent UMETA(DisplayName = "Populate Current Active Agent"),
	PopulateAllAgents UMETA(DisplayName = "Populate All Agents")
};

// Declare the Instance Data and its strucutre. Can be empty. 
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

![statetree-showcase-selected.png](/assets/images/statetree-showcase-selected.png){: .center-image }

Once you have created your StateTree Task you will probably want to override some of its parent functions, especially the EnterState Function. Each of these functions returns a *EStateTreeRunStatus*, which signals the Task and the State what is the result. 

> In Engine version 5.5, when a task completes it will retrigger an state transition, meaning that the first task that finishes in an state will determine the state result, even if other are ongoing. This was changed in version 5.6, making it possible to change on how the state completion is determined. I ended up moving to that version, so I recommend you doing so as well if you can.
> 

This is an example of how our task that tells the current active Agent to pick the die it will throw. 

```cpp
EStateTreeRunStatus FSTRequestActiveAgentDiePick::EnterState(FStateTreeExecutionContext& Context, const FStateTreeTransitionResult& Transition) const
{
	FInstanceDataType& instanceData = Context.GetInstanceData(*this);
	instanceData.bIsWaitingForSelection = true;
	
	if (ADBCombatGameMode* gameMode = Cast<ADBCombatGameMode>(Context.GetOwner()))
	{
		IDBCombatAgentInterface* activeAgent = gameMode->GetCurrentActiveAgent();
		
		if (activeAgent->IsAgentAvailableToPlayTurn() == false)
		{
			// Agent doesn't have Die on its Hand.
			return EStateTreeRunStatus::Failed;
		}
		
		activeAgent->StartDieSelection().AddLambda([this, Context]()
		{
			FInstanceDataType& instanceData = Context.GetInstanceData(*this);
			instanceData.bIsWaitingForSelection = false;
		});
		
		return EStateTreeRunStatus::Running;
	}
	
	return EStateTreeRunStatus::Failed;
}
	
EStateTreeRunStatus FSTRequestActiveAgentDiePick::Tick(FStateTreeExecutionContext& Context, const float DeltaTime) const
{
	FInstanceDataType& instanceData = Context.GetInstanceData(*this);
	
	return instanceData.bIsWaitingForSelection ? EStateTreeRunStatus::Running : EStateTreeRunStatus::Succeeded;
}
```

## Pretty Cool but, Why Bother?

Besides the obvious reason of “It’s cool to learn something new”, I found X particular cases in which I saw that using StateTree’s over other types might be a good idea. 

- **Changing Requirements.** Our project is in a really early stage and it needs quick iteration. This might be true to many other game projects, since Game Devs need to be as reactive as possible whenever we find a better approach. While working in our dice game, I found that we needed to add timers between certain states, and adding a couple of new nodes and a new tasks was all that was needed. This might be better for some games than others, but this seemed like a particular good example for it.
- **Better Access for Designers.** This is a theoretical point as of now, since I haven’t tried it, but it would be idea that a technical designer in the future can modify some values and even complete states from the Game Flow tree. However, even if they don’t, seeing how the flow works in a visual way is at least a simple improvement already.
- **Quick Prototyping.** I see a future in which we want to A/B test a new flow, which will only consist of duplicating the StateTree data asset and changing it there. I can see more complicated flows in which I change the tree at runtime as well.
- **Lower Coupling.** Leaving more high level stuff for the Game Mode to handle and putting specific code in the Tasks makes everything easier to maintain. The structure of the Tasks makes you to maintain the responsibilities of the Game Mode in it, which makes things still go through the Unreal Framework as it should.

## So, everything is good?

As always in Software Development, there is no silver bullet. Even though I think this little experiment was a success there are some things that still don’t quite convince me:

- **Too many Tasks?** At the time of writing, I have 14 tasks already on our project. My approach was to compartmentalize them as much as possible to have multiple of them in a state, and I’m waiting to the Task list to stop growing after the Gameplay stabilizes . Time will tell.
- I’m trying to use StateTrees as they were finite state machines but, as I said at the beginning, they are not meant to be that. They seem to be aimed to facilitate AI, so using them as something they are not meant to be might be problematic in the future. However, as its the closest thing I found to it out of the box, I’ll keep trying to use it as so. We’ll see.

I could do a part two of this post the deeper I go with StateTrees using this approach in case I learn things that change what’s in here. The goal of these posts is to continue learning, so feel free to point out where I screwed up and I’ll update it. 

Cheers!