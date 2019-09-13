# OAuth2 Integration

For simplicity let's assume that our OAuth2 integration is google

## **1(optional).** Add function *ensureAccountCreated* 

```javascript

const ensureAccountCreated = async (socialNetworkName, profile, mapProfile) => {
  const mappedUser = mapProfile(profile);
  let user = await userService.findOne({ email: mappedUser.email });

  if (user) {
    const socialId = user.oauth[socialNetworkName];

    if (!socialId) {
      user = await userService.findOneAndUpdate(
        { _id: user._id },
        { $set: { [`oauth.${socialNetworkName}Id`]: mappedUser.oauth[`${socialNetworkName}Id`] } },
      );
    }

    return user;
  }

  return userService.create(mappedUser);
};

```

## Also add additional middleware *setTokensAndLogin*

```javascript

async function setTokensAndLogin(ctx) {
  await Promise.all([
    userService.updateLastRequest(userId),
    authService.setTokens(ctx, userId),
  ]);

  return ctx.redirect(config.webUrl);
}

```

## **2.** Add mapper for user fields from social network payload

```javascript

const mapGoogleProfile = profile => {
    firstName: profile.given_name,
    lastName: profile.family_name,
    email: profile.email,
    isEmailVerified: true,
    oauth: {
      googleId: profile.sub,
    },
};

```

## **3.** Install and Setup corresponding passport strategy

```javascript

const passport = require('koa-passport'); // TBD - do we need this import
const GoogleStrategy = require('passport-google-oauth').OAuthStrategy;

// Use the GoogleStrategy within Passport.
//   Strategies in passport require a `verify` function, which accept
//   credentials (in this case, a token, tokenSecret, and Google profile), and
//   invoke a callback with a user object.
passport.use(new GoogleStrategy({
    consumerKey: GOOGLE_CONSUMER_KEY,
    consumerSecret: GOOGLE_CONSUMER_SECRET,
    callbackURL: `http://${API_URL}/auth/google/callback`
  },
  async function(token, tokenSecret, profile, done) {
    try {
      // don't forget to import functions ensureAccountCreated and mapGoogleProfile before
      const user = await ensureAccountCreated('google', mapGoogleProfile, profile);
    } catch(err) {
      done(err);
    }

    done(null, user);
  }
));

```

## **4.** Add necessary endpoints in list of your public routes

```javascript
router.get('/auth/google/', passport.authenticate('google'));
router.get('/auth/google/callback', passport.authenticate('google'), setTokensAndLogin); // TBD: need to redirect user in case of error
```

## **5.** Add additional component for auth on website:

```javascript

<a href="${API_URL}/auth/google">Sign in with Google</a>

```