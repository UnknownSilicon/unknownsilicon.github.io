---
title: "ParallelUtils - Part 1: Modular Minecraft Modding Madness"
date: 2026-10-29T15:21:08-04:00
author: "UnknownSilicon"
cover: ""
tags: ["minecraft", "tech", "coding"]
keywords: ["", ""]
description: ""
showFullContent: false
readingTime: false
hideComments: false
table-of-contents: false
toc-auto-numbering: false

draft: false
---

Aat the start of the COVID-19 pandemic, some friends, at the time friends-of-friends, invited me to join a public minecraft server they started called [Parallel](https://parallelmc.org) (also noted as ParallelMC). Parallel is a survival-multiplayer (SMP) server with lore, a narrative, and very welcoming community. At the time, it was more or less a purely vanilla [Paper](https://papermc.io) server. It didn't have custom features and only had a few plugins for QOL and server administration, but they wanted something more to stand out and fit the lore.

The server owner Diamondback (Aka "DB") talked about wanting custom mobs, custom items, and generally more flexibility in what the server did. I offered to help, since I had quite a significant amount of programming experience and had at least looked into Minecraft modding in the past. And so, what would become known as [ParallelUtils](https://github.com/ParallelMC/ParallelUtils) (PU) was born.

# The Wisp

PU was initially intended to only add custom mobs. At the time, there weren't easy APIs or free plugins that could provide this functionality. Existing solutions were paid, inflexible, or simply not what we needed. The goal here was simple: First, create a single custom mob (a "wisp"; A neutral mob made of cloud particles that dropped special items upon death) to prove it works, then create a framework around this to facilitate adding more.

{{< img class=center-fig src=/wisp.png width=500px alt="This image depicts an in-game wisp. It is a puff of cloud particles. The wisp is next to portal blocks and standing on a floating island in the sunset." caption="A wisp">}}

This was certainly not my proudest work, but I learned a lot about the internals of Minecraft servers, the associated toolchains, and working with obfuscated code. Minecraft/Paper 1.19 did not support custom mobs natively. The best we had was setting attributes via NBT (health, speed, invisible, etc.) In order to change behavior/pathfinding, we had to use "NMS" and Java reflection. "NMS" stands for Net Minecraft Server, used as a colloquial term for anything sitting in the `net.minecraft` Java package, which contains the internal server functionality. This very often involves accessing private methods through reflection and touching sensitive APIs.

For a simple example, the following code block shows what's needed to clear the pathfinding goals and targets from a given Mob. Thankfully there's now an official API for this, but reflection nonsense has been unfortunately commonplace in Minecraft modding. Even today, other functionality/features still need reflection to access. At the time, Minecraft internals were still obfuscated. [Community tools](https://mcsrc.dev/) existed to map between obfuscated Minecraft, and the de-obfuscated versions different projects used (Force, Fabric, and Bukkit all had their own mappings that differed slightly). Most of these tools no longer exist since Mojang now provides mappings, or entirely unobufuscated code.

{{< code title="Reflection Example" >}}
```java
	/**
	 * Clears the pathfinding goals for an entity
	 * @param entity The entity to clear the pathfinding goals of
	 */
	public static void clearGoals(Mob entity) {

		Map goalC = (Map) getPrivateField("c", GoalSelector.class, entity.goalSelector);
		goalC.clear();
		Set goalD = (Set) getPrivateField("d", GoalSelector.class, entity.goalSelector);
		goalD.clear();
		Map targetC = (Map) getPrivateField("c", GoalSelector.class, entity.targetSelector);
		targetC.clear();
		Set targetD = (Set) getPrivateField("d", GoalSelector.class, entity.targetSelector);
		targetD.clear();
	}

	/**
	 * Returns the object associated with a private field of an object using reflection.
	 * @param fieldName The name of the private field to get
	 * @param clazz The class containing the private field
	 * @param object The object to pull the private field from
	 * @return Returns the object associated with the given private field
	 */
	public static Object getPrivateField(String fieldName, Class clazz, Object object) {
		Field field;
		Object o = null;

		try {
			field = clazz.getDeclaredField(fieldName);

			field.setAccessible(true);

			o = field.get(object);
		} catch (NoSuchFieldException | IllegalAccessException e) {
			e.printStackTrace();
		}
		return o;
    }
```
{{< /code >}}

Pathfinding goals and targets are how Mobs decide what to do. Goals are an ordered list of tasks to complete, while targets are the entities or locations the mob performs certain tasks on. Wisps goals are simple:
1. Attack Targets
2. Random Stroll

`MeleeAttackGoal` doesn't do anything without a target, so wisps get `HurtByTargetGoal`. This goal makes wisps only attack entities that attack them, a "neutral" mob. We add these goals, some particles, and setup attributes and drops and we're good to go.

{{< code title="Wisp Codes" >}}
Sidenote: This code is bad and old, but it's here for historical reference. It's been rewritten a bit since then, but it's less hacky and therefore less interesting imo.

```java
//...
// Add a Cloud particle effect for wisps with specific amount and timing parameters
ParticleRegistry.getInstance().registerParticles("wisp", new ParticleOptions(Particle.CLOUD, 50, 0.5, 1, 0));
//...

// Wisps are based on Zombies, mostly arbitrarily
public class EntityWisp extends Zombie {
	public EntityWisp(EntityType<? extends Zombie> entitytypes, Level world) {
		super(entitytypes, world);
		registerGoals();
	}

	public EntityWisp(Level world) {
		super(world);
		registerGoals();
	}

	@Override
	public void registerGoals() {
		initPathfinder(this);
	}

	public static void initPathfinder(Zombie zombie) {
		//clearing Zombie goals
		CustomEntityHelper.clearGoals(zombie);

		// Wisps first attack their target if they have one, then randomly walk around
		zombie.goalSelector.addGoal(0, new MeleeAttackGoal(zombie, 1.0, false));
		zombie.goalSelector.addGoal(1, new RandomStrollGoal(zombie, 1.0));

		zombie.targetSelector.addGoal(0, new HurtByTargetGoal(zombie));
	}

	public static EntityWisp spawn(JavaPlugin plugin, CraftServer server, CraftWorld world, Location l) {
		return spawn(plugin, server, world, l, SpawnReason.UNKNOWN, null);
	}

	public static EntityWisp spawn(JavaPlugin plugin, CraftServer server, CraftWorld world, Location l,
	                               SpawnReason reason, Location origin) {
		// FIrst, create and spawn a zombie, then set up the Wisp internal parameters for it
		EntityWisp wisp = new EntityWisp(world.getHandle());
		CraftZombie zombie = (CraftZombie) CraftEntity.getEntity(server, wisp);

		setup(plugin, zombie);
		wisp.setPos(l.getX(), l.getY(), l.getZ());
		boolean spawned = world.getHandle().addFreshEntity(wisp, CreatureSpawnEvent.SpawnReason.CUSTOM);

		if (!spawned) {
			ParallelUtils.log(java.util.logging.Level.INFO, "Unable to spawn entity");
			return null;
		}

		// Save state to ensure custom mobs remain after server restarts
		EntityRegistry.getInstance().registerEntity(zombie.getUniqueId().toString(), "wisp", wisp, reason, origin);

		return wisp;
	}

	public static Zombie setup(JavaPlugin plugin, CraftZombie mob) {
		CraftWisp.setupNBT(mob);

		Zombie wisp = mob.getHandle();

		EntityWisp.initPathfinder(wisp);

		// Start particle task if it isn't already running
		if (!ParticleRegistry.getInstance().particleTaskRunning) {
			BukkitTask task = new ParticleTask().runTaskTimer(plugin, 0, 10);
			ParticleRegistry.getInstance().particleTaskRunning = true;
		}

		return wisp;
	}

	@Override
	public boolean alwaysAccepts() {
		return super.alwaysAccepts();
	}
}

public class CraftWisp {

	/**
	 * Setup the NBT data for a custom Wisp, represented with a Zombie entity
	 * @param entity The entity to setup the NBT for
	 */
	public static void setupNBT(CraftZombie entity) {
		// Attributes
		// Health = 20
		entity.getAttribute(GENERIC_MAX_HEALTH).setBaseValue(20.0);
		entity.setHealth(entity.getAttribute(GENERIC_MAX_HEALTH).getBaseValue());

		// Damage = 8
		entity.getAttribute(GENERIC_ATTACK_DAMAGE).setBaseValue(8.0);

		// Follow range = 16
		entity.getAttribute(GENERIC_FOLLOW_RANGE).setBaseValue(16.0);

		// Movement speed = 0.25
		entity.getAttribute(GENERIC_MOVEMENT_SPEED).setBaseValue(0.25);

		// No zombie reinforcements
		entity.getAttribute(ZOMBIE_SPAWN_REINFORCEMENTS).setBaseValue(0);

		// Other stuff

		// Invisible
		entity.setInvisible(true);

		// Can't pickup items
		entity.setCanPickupItems(false);

		// No random equipment
		entity.getEquipment().clear();

		// Don't burn in sunlight
		entity.setShouldBurnInDay(false);

		// Silent
		entity.setSilent(true);
	}

	private static ItemStack shard;

	/**
	 * Handle the EntityDeathEvent for a Wisp.
	 * Drop a Soul Shard
	 * @param event The EntityDeathEvent to handle
	 */
	public static void deathLoot(EntityDeathEvent event) {
		// The shard only needs to be made once
		if (shard == null) {
			shard = new ItemStack(Material.PRISMARINE_SHARD, 1);

			try {
				ItemMeta shardMeta = shard.getItemMeta();
				TextComponent name = Component.text("Soul Shard", NamedTextColor.WHITE)
						.decoration(TextDecoration.ITALIC, false);
				shardMeta.displayName(name);
				shardMeta.setCustomModelData(1000001); // Show custom texture
				shardMeta.addEnchant(Enchantment.DURABILITY, 1, true); // Make it look enchanted
				shardMeta.addItemFlags(ItemFlag.HIDE_ENCHANTS);

				PluginManager manager = Bukkit.getPluginManager();
				JavaPlugin plugin = (JavaPlugin) manager.getPlugin(Constants.PLUGIN_NAME);
				if (plugin == null) {
					ParallelUtils.log(Level.SEVERE, "PLUGIN NOT FOUND. THIS IS A PROBLEM");
					return;
				}

				// A special tag to keep track of custom items
				NamespacedKey key = new NamespacedKey(plugin, "CustomItem");
				shardMeta.getPersistentDataContainer().set(key, PersistentDataType.INTEGER, 1);
				shard.setItemMeta(shardMeta);
			} catch (NullPointerException e) {
				e.printStackTrace();
			}
		}

		List<ItemStack> drops = event.getDrops();
		drops.clear();
		drops.add(shard);
		event.setDroppedExp(0);
	}
}
```
{{< /code >}}

# Wrap Up

Custom Mobs has changed a lot since it's origin, mostly due to more accessible programming interfaces, that have allowed it to become a lot cleaner and more direct. This is not the only approach for this either. With a similar internal registry hack to what I'll talk about in the future, it's possible to add a new mob type that the server natively understands, allowing for even smoother compatability, as opposed to the current solution that must be based on existing entities. 

In the next few posts of this series, I'll be talking about the modular loading system I developed to allow hotloading individual plugin components, a dynamic and data driven "Charms" system to add special effects that can be applied to items, and a semi-secret project to push the boundaries of what's possible with a Paper server and a vanilla client. 