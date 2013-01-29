---
layout: post
title: "Facebook profile pictures in cocos2d"
date: 2013-01-28 19:32
comments: true
categories: [iOS, Mobile, cocos2d, Facebook, Objective-C]
---

## Facebook and UIKit

Somewhere along the line of developing Factor Friends (more on that later) I decided that I wanted, no, _needed_ Facebook integration.  
Despite whether or not this was a necessary feature was trumped by my desire to play around with social networking APIs.  
If you haven't read through their tutorial yet, I strongly suggest you [start here](https://developers.facebook.  com/docs/tutorials/ios-sdk-tutorial/).

For the most part, integration with the Facebook API and cocos2d is straight forward.  
The data model is separated from any framework and the view controllers for login are dealt with for you.  
Where it becomes tricky is when you need to bridge the gap between cocos2d and UIKit, which is what the Facebook API is built on.  

The normal way one would construct a profile image from the API is as follows:

```objective-c Making a profile picture in UIKit
// Is the player logged into Facebook?  If so load their Facebook profile pic.
if ([FBSession activeSession].isOpen) {
	[[FBRequest requestForMe] startWithCompletionHandler:^(FBRequestConnection *connection, NSDictionary<FBGraphUser> *user, NSError *error) {
		if (!error) {
			FBProfilePictureView *pic = [[FBProfilePictureView alloc] initWithProfileID:@"my_facebook_id" pictureCropping:FBProfilePictureCroppingSquare];
			CGSize s = [CCDirector sharedDirector].winSize;
			CGRect f = pic.frame;
			// Center the profile picture in the screen.
			f.origin = ccp(s.width / 2, s.height / 2);
			pic.frame = f;
			[[CCDirector sharedDirector].view addSubview:pic];
		}
	}];
}
```

## cocos2d and UIViews

Cool!  With any luck, we have our Facebook profile picture sitting in the middle of the screen.  
In some cases, this might be enough.  
Where it started to break for me was when I needed profile pictures that _moved_ with respect to the rest of my cocos2d scene, and behaved like normal `CCSprite`s.  
There are some situations where it is just flat out easier to deal with a `CCNode` subclass than to manipulate a `UIView`.  
In this case, an `FBProfilePictureView` would not cut it, since I needed access to the underlying data.  
After attempting to extract the underlying `UIImageView` and tamper with the CGImage:

```objective-c Do not attempt
FBProfilePictureView *pic = ...;
for (id child in pic) {
	if ([child isKindOfClass:[UIImageView class]]) {
		// ref is nil.
		CGImageRef ref = ((UIImageView *)child).image.CGImage;
		// Will throw an error.
		CCTexture2D *tex = [[CCTexture2D alloc] initWithCGImage:ref resolutionType:kCCResolutioniPhone];
	}
}
```

Needless to say, that did not work.  
Someone who has more experience than me might be able to figure it out, but I conceded defeat.  
I came to the conclusion that I would probably have to navigate away from the Facebook iOS framework and use their web API directly.

## Facebook Graph API

Luckily for me, accessing profile pictures directly is rather easy.
The full documentation for this can be found [here](https://developers.facebook.com/docs/reference/api/using-pictures/).
To test it out, try navigating to:

__https://graph.facebook.com/your_id/picture__

where __your_id__ is either your Facebook ID or your Facebook 'address'.
For instance, you can view this ugly guy here:

[https://graph.facebook.com/paul.andrewharrison.moore/picture](https://graph.facebook.com/paul.andrewharrison.moore/picture)
{% img center https://graph.facebook.com/paul.andrewharrison.moore/picture "Me" %}

We can also pass parameters to this URL to customize the properties of the image.
For instance, we can adjust the width and height of the image:

[https://graph.facebook.com/paul.andrewharrison.moore/picture?width=60&height=120](https://graph.facebook.com/paul.andrewharrison.moore/picture?width=60&height=120)  
{% img center https://graph.facebook.com/paul.andrewharrison.moore/picture?width=60&height=120 "Me" %}

Read the documentation for a full discussion on what other options are available.  

## Downloading remote images

Anyways, the issue now is how to download this picture and use it as a `CCSprite`.  
The general formula looks something like this:

```objective-c Downloading a remote image in Objective-C
NSString *facebookId = ...;
int width = ...;
int height = ...;
NSString *https = [NSString stringWithFormat:@"https://graph.facebook.com/%@/picture?width=%i&height=%i", facebookId, width, height];
NSURL *url = [NSURL URLWithString:https];
NSAssert(url, @"URL is malformed: %@", https);
NSError *error = nil;
NSData *data = [NSData dataWithContentsOfURL:url options:0 error:&error];
if (data && !error) {
	// Download the image to the Cache directory of our App.
	NSString *path = [[NSSearchPathForDirectoriesInDomains(NSCachesDirectory, NSUserDomainMask, YES) lastObject] stringByAppendingPathComponent:[facebookId stringByAppendingString:@".jpg"]];
    [data writeToFile:path options:NSDataWritingAtomic error:&error];
    if (!error) {
        // It worked!
    } else {
        // Error, Could not save downloaded file!
    }
} else {
    // Error, Could not download file!
}
```

This will download our Facebook profile image to the Cache directory as a JPEG file with our Facebook ID as the name.  
To use this image as a `CCSprite`, we must first load the image into a `CCTexture2D` and create a sprite frame, like so:

```objective-c Using the Cached image as a CCSprite
// 1. Load the texture in from the Cache directory.
CCTexture2D *texture = [[CCTextureCache sharedTextureCache] addImage:path];
// 2. Create the sprite frame with appropriate dimensions.
CCSpriteFrame *frame = [CCSpriteFrame frameWithTexture:texture rect:CGRectMake(0, 0, width, height)];
// 3. Golden.
[CCSprite spriteWithSpriteFrame:frame];
```

You may be thinking, cool, but this is tedious to do each time, and...

1. This code runs synchronously, wouldn't my App block while waiting for the picture to download?
2. If I've downloaded the image once, can't I just reload it next time from the cache?
3. While the image is downloading, can I display a placeholder image?
4. Can this functionality be wrapped in a `CCSprite` subclass?

Yes you can!  I've done so myself, and you can find it in [this Github repository](https://github.com/paulmoore/CCFBProfilePicture).  
It's nothing spectacular, but it certainly gets the job done for my purposes.  
Here is an example of how to use it:

```objective-c Using the CCFBProfilePicture class
CCFBProfilePicture *pic = [CCFBProfilePicture profilePictureWithId:fbid accessToken:accessToken contentSize:content cached:useCache placeholder:temp];
[self addChild:pic];
```

The class downloads the image in a background thread, so your App will not stall.  
It also (optionally) attempts to use cached versions of the image before querying Facebook.  
As well, you can specify a placeholder sprite frame which is displayed while the image is loading.  

The constructor requires these parameters:

* __`fbid`__ Your Facebook ID, e.g. _paul.andrewharrison.moore_
* __`accessToken`__ (Optional) This is the Facebook access token you received when you signed in through your App.  Providing this argument reduces the chance your App will be rate limited (limit the amount of profile pictures you can access).
* __`content`__ The size, in points, to download the image at.  For instance, if you specify `CGSizeMake(100, 100)` and you are running on an iPhone 4, it will download an image 200x200 pixels big.  This will also be the `contentSize` property of the sprite.
* __`useCache`__ If set to YES, will attempt to use a cached version of the image before downloading a new one.  This is a recommended option unless you must always have an up-to-date image, or you need multiple resolutions of the image.
* __`temp`__ (Optional) This is the sprite frame that will be displayed while the image is loading.  It is recommended that it is the same content size as the downloaded image.  

And that is it!  Now you can run actions on the profile pictures and place them in `CCScrollViews` and whatnot.  
Feel free to contribute or report any bugs.  

{% img center /images/posts/fbprofilepic.png "Blah" "Facebook in Factor Friends" %}

---

### References

* [Background loading a Cocos2d Sprite from URL](http://srooltheknife.blogspot.ca/2012/05/background-loading-cocos2d-sprite-from.html)
