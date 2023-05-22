# sbox-projectiles
🚀 Lag compensated simulated projectiles using game resources for projectile definitions.

# Projectile Definitions
You can define projectile definition assets by right clicking in the asset browser and creating a new `Projectile` asset.

![Image](https://files.facepunch.com/conna/1b2211b1/sbox-dev_opc1W1G2RX.png)

## Life Time
This is the time (in seconds) that a projectile will stay alive.

##  Gravity
How much gravity will affect the projectile. The higher the number, the faster it will reach the ground. Use `0` for projectiles that aren't affected by gravity.

## Speed
The speed of the projectile in units per second.

## Radius
The size of the projectile (width of the trace). A larger value will result in wider hits.

## Model Name
The model to use for the projectile (not required.)

## Explosion Effect
When a projectile hits something this is the particle effect that will be used.

## Trail Effect
If defined this particle effect will be attached to the projectile.

## Launch Sound
A sound to play when the projectile is launched.

## Hit Sound
A sound to play when the projectile hits something.

# ProjectileSimulator
If the projectile is to be fired by a player, then each player's pawn needs to have a `ProjectileSimulator` and its `Simulate` method needs to be called at some point inside of the pawn's `Simulate` method.

## Example Usage

```csharp
public ProjectileSimulator Projectiles { get; private set; }

public MyPawnEntity() : base()
{
	Projectiles = new( this );
}

public override void Simulate( IClient client )
{
	base.Simulate( client );
	Projectiles.Simulate();
}
```

# Creating Projectiles
When fired by a player, projectiles should be created in a predicted context and as such should be created both on the firing client and on the server. The best way to do this is simply within the player's `Simulate` method (for example when they have pressed an attack button.)

You should also call `Game.SetRandomSeed( Time.Tick )` before you begin firing any projectiles in this way (predicted.) Here's an example that you could use in a weapon.

```csharp
public override void AttackPrimary()
{
	if ( Prediction.FirstTime )
	{
		Game.SetRandomSeed( Time.Tick );
		FireProjectile();
	}
}

private void FireProjectile()
{
	if ( Owner is not MyPawnEntity player )
		return;

	if ( string.IsNullOrEmpty( ProjectileData ) )
	{
		throw new Exception( $"Projectile Data has not been set for {this}!" );
	}

	var projectile = Projectile.Create<T>( ProjectileData );

	// Don't hit this weapon entity.
	projectile.IgnoreEntity = this;

	// We need to set the simulator to use (as mentioned earlier.)
	projectile.Simulator = player.Projectiles;

	// The attacker is the player who owns this weapon.
	projectile.Attacker = player;

	// Fire from the muzzle attachment on the weapon (this is just an example.)
	var muzzle = GetAttachment( "muzzle" );
	var position = muzzle.Value.Position.WithZ( MathF.Max( muzzle.Value.Position.z, player.EyePosition.z ) );
	var forward = player.EyeRotation.Forward;
	var endPosition = player.EyePosition + forward * BulletRange;
	var trace = Trace.Ray( player.EyePosition, endPosition )
		.Ignore( player )
		.Ignore( this )
		.Run();

	var direction = (trace.EndPosition - position).Normal;
	direction += (Vector3.Random + Vector3.Random + Vector3.Random + Vector3.Random) * Spread * 0.25f;
	direction = direction.Normal;

	// Let's fetch the speed value from the projectile data, but we could use any value or modify it.
	var speed = projectile.Data.Speed.GetValue();
	var velocity = (direction * speed) + (player.Velocity * InheritVelocity);
	projectile.Initialize( position, velocity, OnProjectileHit );
}

private void OnProjectileHit( Projectile projectile, Entity target )
{
	if ( Game.IsServer && target.IsValid() )
	{
		// We hit something.
	}
}
```
