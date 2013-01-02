# SpringBoard Color Sort
__Sorts iOS icons by color__

## Prerequisites
Requires a jailbroken iOS device (tested with iOS 5) with OpenSSH server installed.
This probably won't work well with other tools that mod the springboard.

## Experimental Usage
Open `Rakefile` and set the `USER` and `HOST` variables accordingly.

    bundle install
    rake
  
## Todo
* Better color detection, only looking at borders of icons
* Better color sorting, taking into account saturation in addition to hue
* Handle system apps

## Usable?
Not yet.