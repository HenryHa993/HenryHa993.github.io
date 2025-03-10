---
title: "HECS - Entity Component System"
description: "My homemade ECS system, built using C++. Currently in-progress."
publishDate: "26 Feb 2025"
coverImage:
    src: "./hecs-cover.png"
    alt: "HECS Cover Image"
coverGif:
    src: "./hecs-cover.png"
    alt: "Hecs Cover Image"
tags: ["c++", "ECS", "tools-dev","in-progress"]
draft: false
relatedPosts: []
---

## About

Henry's Entity Component System (HECS) is my attempt to make my own home-grown ECS framework. ECS implementation is a pretty big endeavour, so I wanted to do some writing to help me think more deeply about the details. This project is currently in-progress.

## Motivation

My personal motivation behind developing HECS is to build a better understanding of Entity Component Systems as well as delve into some C++ features I was interested in. Implementing an ECS typically requires template metaprogramming, building memory pools and iterators -- all topics I am interested in.

## Implementation

I decided to structure this section by the problems I ran into.

### Overview

Note: With so much I had to learn to even start this project, I decided to build many aspects ad-hoc. This gave me a bit of flexibility to adapt and experiment, so implementation isn't final.

HECS aims to store components continuously in memory, allowing for efficient cache utilisation in certain cases. It also aims to keep the implementation of entities as simple as possible: entities are simply used as an index for the components that they own. This was purposefully done to streamline both the implementation and my understanding of the topic.

From a design perspective, HECS also separates data from implementation. For me, this was the most appealing aspect of ECS.

### Storing Components

How an ECS stores components is one of the fundamental problems in developing one. This is because how you store components can have different implications on how you add/remove components from entities, how you delete entities and how you iterate through components for systems.

In researching, I was overwhelmed by the number of possible implementations. One implementation trades memory size and speed for the design opportunities of ECS by storing components in memory pools and using entities to index them [^1]. This leaves potential gaps in component pools, as each element is reserved for a specific entity. This is a less common approach for full-scale ECS implementations due to those costs, but I found the post to be a great starting point for developing my own.

[^1]: [How to make a simple entity-component-system in C++](https://www.david-colson.com/2020/02/09/making-a-simple-ecs.html), David Colson

I found that the two main ways to store components was either through archetypes or sparse sets.

Archetypes is an approach used by frameworks such as FLECS, which groups components in memory by the type of component structure (archetype) an entity has, rather than the components themselves [^2]. This was an approach I avoided, as I would have to account for changes in an entity's archetype by moving data across archetype component pools. This adds a few non-trivial challenges to the implementations and not necessarily ones I wanted to tackle with my baby ECS.

[^2]: [Building an ECS #2: Archetypes and Vectorization](https://ajmmertens.medium.com/building-an-ecs-2-archetypes-and-vectorization-fe21690805f9), Sanders Mertens

Sparse Sets adds a level of indirection to the first implementation I listed. A sparse set maintains both a sparse array and packed array. The sparse array contains pointers or indexes to the entity's component in the packed array. This approach makes iterating through individual components blazingly fast and is used by frameworks such as ENTT. There's a great post from the creator of ENTT about the subject [^3].

[^3]: [ECS Back and Forth Part 2 - Where are my entities?](s://skypjack.github.io/2019-03-07-ecs-baf-part-2/), Michele **skypjack** Caini

Ultimately, I decided on an approach which uses independent sparse sets as components pools. Initially, I wanted to go with the first approach, but an aspect of ECS I thought was interesting to engage with is storing the components continuously in memory.

Now on to my actual implementation! To start, for every component there is a sparse set.

```cpp
template <class T>
class ComponentPool : public ISparseSet
{
	...
	
	unsigned NumComponents;
	
	unsigned SparseArray[MAX_ENTITIES];
	unsigned EntityPackedArray[MAX_ENTITIES];
	T ComponentPackedArray[MAX_ENTITIES];
	
};
```

Each component pool inherits from a common `ISparseSet` interface, providing basic functions for the ECS manager to interact with. This also utilises polymorphism to be able to store component pools together. The class also utilises templates, and the user will be able to register their components to the ECS manager.

Another interesting detail here is that it maintains three different arrays rather than the mentioned two. When using sparse sets, in order to verify the owner of the packed array element we need to store the ID in the packed array. We check it as so:

```cpp
bool Has(unsigned entity) override
{
	return
		SparseArray[entity] != UINT_MAX
		&&
		EntityPackedArray[SparseArray[entity]] == entity;
}
```

Originally, I had a wrapper around the component to store the owning entity. I later realised that doing this would increase the size of each element in the packed array, taking up memory which could be used for other components when filling up the cache line.

Another interesting implementation detail is how you remove entities from the sparse set:

```cpp
void Remove(unsigned entity)
{
	// Get packed array index of component being removed
	unsigned index = SparseArray[entity];

	NumComponents--;

	// Swap component and entity with the last element in array
	T& component = ComponentPackedArray[index];
	component = ComponentPackedArray[NumComponents];
	
	unsigned& entityAddress = EntityPackedArray[index];
	entityAddress = EntityPackedArray[NumComponents];
	
	SparseArray[entityAddress] = index;
	SparseArray[entity] = UINT_MAX;
}
```

When removing entity components, the sparse array swaps the last element in the packed array with the element being deleted. Relevant indexes are then updated, keeping the entity and component arrays nice and packed.

### Storing Pools of Components

Another challenge I came across was how to store component pools of different types in the same data structure. When learning about templates, I read that it was useful to think of it as "compile-time polymorphism". Combining both compile and runtime polymorphism, we can register components and proceed to operate on them in the ECS manager.

```cpp
class World{
	template <class T>
	void Register()
	{
		unsigned componentId = IDGenerator::GetID<T>();
		ComponentPools.push_back(std::make_unique<ComponentPool<T>>());
		
		UE_LOG(LogTemp, Warning, TEXT("Component registered with ID %u"), componentId)
	}

...

	std::vector<std::unique_ptr<ISparseSet>> ComponentPools;
}
```

### Generating Component IDs

Once I had a data structure containing different component pools, I needed a way to access them for a given component. **skypjack** mentions a nifty way to do this combining templates and static types [^4].

[^4]: [ECS Back and Forth Part 1 - Introduction](https://skypjack.github.io/2019-02-14-ecs-baf-part-1/), Michele **skypjack** Caini

```cpp
/*
 * Generates an ID for each new type passed in
 */
class IDGenerator
{
	static unsigned Identifier()
	{
		static unsigned value = 0;
		return value++;
	}

public:
	template<class T>
	static unsigned GetID()
	{
		static const unsigned value = Identifier();
		return value;
	}
};
```

### Systems

At this stage, systems are a work in-progress. In order to get the design concepts of ECS working, I made a simplistic system which iterates through entities, checks their components and runs the behaviour.

Here is an example of a system which accesses two components. I chose this one, because there are a lot of improvements to discuss here.

```cpp
void ASystemsManager::CosXSystem(float deltaTime)
{
	for(unsigned entity = 0; entity < ECS->GetNumEntities(); entity++)
	{
		if(!ECS->Has<CosX>(entity) || !ECS->Has<StaticMeshComponent>(entity))
		{
			continue;
		}

		StaticMeshComponent& meshType = ECS->Get<StaticMeshComponent>(entity);
		CosX& cosXType = ECS->Get<CosX>(entity);
		
		UStaticMeshComponent* mesh = meshType.Value;
		float radius = cosXType.Radius;
		
		FVector newLocation = mesh->GetRelativeLocation();
		newLocation.X = radius * FMath::Cos(GetWorld()->GetTimeSeconds());

		mesh->SetRelativeLocation(newLocation);
	}
}
```

There is lots weaknesses here. We are iterating through entities, and going through lots of indirection (due to sparse sets) in order to check for their components. This is definitely not final, and future implementations will focus on iterating through the components themselves.

## Future Work

If you have read this far, thank you for reading . This has been a really fun project to start, and I am learning so much from it.

Once again, this project is not finished and I will continue to update it. Here are a couple of plans I have for the future.

1. Implementing a scene view to make systems crafting more intuitive
2. Variadic template packs for scene views
3. Making systems more efficient for accessing more than one component
4. Performance measures

Thanks for reading!

## Related Posts