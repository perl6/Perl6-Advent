# Rudolph on Raku

## finding a way home to the North pole with Physics::Navigation

So, Rudolph has been worried about getting Santa and the other reindeer back home to the North Pole 
after an exhausting flight to visit all the (well-behaved) Children on the Globe.

He has heard a rumour that the North Pole keeps moving due to the precession of molten iron at 
the Earth's core and that every year it creeps around a bit with relation to Santa's workshop,
which lies at the True North Pole.

Luckily he has been on a navigation skills course and has learned about how to specify a position on
the globe using a combination of Latitude and Longitude. However, these seem all of a muddle as they 
are alike and yet different. What Rudi needs is a way to structure his navigation to ensure that he 
does not mix them up. Even better, he is good friends with Larry and knows that he can trust the 
Raku type system to get him home.

Let's see how he does it:

```
use Physics::Unit;
use Physics::Measure;

class NavAngle is Angle {
  has Unit  $.units is rw where *.name eq '°';
	
  multi method new( Str:D $s ) {						say "NA new from Str" if $db; 
    my ($decimal, $compass) = NavAngle.defn-extract( $s );
    my $type;
    given $compass {
      when <N S>.any   { $type = 'Latitude' }
      when <E W>.any   { $type = 'Longitude' }
      when <M T H>.any { $type = 'Bearing' }
      default          { nextsame }
    }
    ::($type).new( value => $nominal, compass => $compass );
  }
  
  method defn-extract( NavAngle:U: Str:D $s ) {
    #handle degrees-minutes-seconds <°> is U+00B0 <′> is U+2032 <″> is U+2033
    unless $s ~~ /(\d*)\°(\d*)\′?(\d*)\″?\w*(<[NSEWMTH]>)/ { return 0 };
    my $deg where 0 <= * < 360 = $0 % 360;
    my $min where 0 <= * <  60 = $1 // 0;
    my $sec where 0 <= * <  60 = $2 // 0;
    my $decimal = ( ($deg * 3600) + ($min * 60) + $sec ) / 3600;
    my $compass = ~$3;

    say "NA extracting «$s»: value is $deg°$min′$sec″, compass is $compass" if $db;
    return($decimal, $compass)
  }
  
  method Str {
    my ( $deg, $min ) = self.dms( :no-secs ); 
    $deg = sprintf( "%03d", $deg );
    qq{$deg° $.compass}
  }
}
#real code at https://github.com/p6steve/raku-Physics-Navigation/blob/master/lib/Physics/Navigation.rakumod
```
So Rudi has inherited the Angle type provided by Physics::Unit and created some general methods that 'know' that 
N S are Latitude and E W are Longitude. There's also the notion of M T H for Bearing (more on that later).

Having attended the Greenland Grammar school, he knows that the Raku regex capability and unicode support
can make short work of degrees, minutes and seconds. Constraints will stop him from flying off at 451 degrees.

Now he can define the Latitude, Longitude and Bearing classes:
```
class Latitude is NavAngle is export {
	has Real  $.value is rw where 0 <= * <= 90; 
	has Str   $.compass is rw where <N S>.any;
}
class Longitude is NavAngle is export {
	has Real  $.value is rw where 0 <= * <= 180; 
	has Str   $.compass is rw where <E W>.any;
}
```
See how the constraints are adjusted to reflect the limits of each child class.

Now Rudolph can set his position:

```my $lat1 = Latitude.new( value => 45, compass => <N> ); say ~$lat1; #OUTPUT 43° N```

But this is quite long winded and he is impatient to get home. Raku unicode and operator overrides come
to the rescue:
```
#define the pisces operator to declare and initialise a new instance 
multi infix:<♓️> ( Any:U $left is rw, Str:D $right ) is equiv( &infix:<=> ) is export {
    $left = NavAngle.new( $right );
}
```
Now he can quickly hoof in his coordinates:
```
my $lat  ♓️ <55°30′30″S>; say ~$lat;   #OUTPUT 55°30.5 S
my $long ♓️ <45°W>;       say ~$long;  #OUTPUT 45° W
```
# Magnetic vs. True North