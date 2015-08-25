WordPress is one of the most easy-to-install and easy-to-use PHP applications available. It's so popular that, as of this month, WordPress powers [nearly 25%](http://w3techs.com/technologies/details/cm-wordpress/all/all) of _all websites on the Internet_.

This means WordPress is fairly well-supported for developers. Unfortunately it also means that WordPress is a large target for potential hackers. Luckily, there are steps all developers can follow to secure their WordPress installation.

## Security Comes First

Like any application, WordPress' largest security vulnerability isn't in the code running on the server or the stack running under the hood: the biggest threat to WordPress is the users that interact with it.

### 1. Only Create Strong Passwords

A vault is only as secure as the lock on the door - a WordPress site is only as secure as the passwords that protect its user accounts.

Since version 3.7, WordPress has shipped with [a password strength meter](http://wptavern.com/ridiculously-smart-password-meter-coming-to-wordpress-3-7) powered by Dropbox's [zxcvbn](https://blogs.dropbox.com/tech/2012/04/zxcvbn-realistic-password-strength-estimation/) library. Thanks to this tool, you can ensure that all of your passwords are sufficiently complex enough to resist automated password crackers.

### 2. Use a Password Manager

A secure password, however, can be immediately rendered _insecure_ depending on how you use it. Password should _never_ be used for multiple logins. Instead, use a unique, strong password for every login you use online.

This goes beyond WordPress - every online account should have its own password specifically to protect against any one service's hack inadvertently opening the door to another service you use.

Tools like [1Password](https://agilebits.com/onepassword) or [Lastpass](https://lastpass.com/) can track multiple passwords for you in a completely secure manner.

### 3. HTTPS Everywhere

If your site isn't already encrypted with SSL, it should be.

Every time you access your site, your browser sends an authentication cookie along with the request to the server. If someone were to intercept this request, they could use the same cookie to impersonate _you_ on the server and execute commands on your behalf.

Installing an SSL certificate on your server is quick and inexpensive; most hosts will take care of it for you! Once you're browsing your site (both the front-end and admin) from an `https://` URL, every request is protected from prying eyes.

### 4. Never Trust Anyone

Likewise, you should make every effort to protect any and _all_ of your online communications from eavesdroppers. Manage your site from a Starbucks? Edit content from a hotel? Moderate comments from a friend's house out-of-town?

These are all incredibly easy ways to get your site hacked at the end of the day.

To better protect your identity, your content, and the integrity of your site, only ever manage your site over a trusted Internet connection. Only connect from home, a tethered connection, or a VPN.

## User Management

Whereas the above advice could apply to literally _any_ online service, the following WordPress-specific tips are deeply rooted in WordPress' architecture and the design of the application code.

### 5. One User, One Role

A common mistake when managing WordPress is using a single user account for everything - often this is the first and only account you've created. Under the hood, though, WordPress implements [a sophisticated role and permissions system](https://codex.wordpress.org/Roles_and_Capabilities) that enables purpose-specific user accounts.

**In short,** do not use your adminsitrator account unless you're modifying adminstrative settings within the website.

- An Contributor role exists for creating (but not publishing) content
- An Author role exists for creating and publishing content
- An Editor role exists for creating, publishing, and editing the content of others

The Administrator role exists to manage the site as a whole - this includes updating plugins and the theme, changing sitewide settings, creating and managing user accounts, _and even editing the code that runs in production_. Only use accounts with this level of privilege when absolutely necessary (as to limit the potential damange wreaked if someone compromises your active account).

### 6. Use Capabilities Responsibly

Be sure you understand the existing [roles and capabilities](https://codex.wordpress.org/Roles_and_Capabilities) system, and use it accordingly. WordPress ships with functions that allow you to restrict certain portions of the site to users with specific capabilites.

For example, say you add an `Advertisement` management page to WordPress, and you want to lock this down to just users with the "Ad Manager" role:

```php
// First, add a custom "Ad Manager" role
function add_ad_manager() {
    add_role( 
        'ad-manager', 
        __( 'Ad Manager', 'locale' ), 
        array( 
            'read'       => true, 
            'manage-ads' => true,
        )
    );
}
add_action( 'init', 'add_ad_manager' );
```

Now, on any page you want to lock down to _just_ Ad Managers, you add a conditional check:

```php
if ( current_user_can( 'manage-ads' ) ) {
    // ... Include a template or use custom logic here.
} else {
    wp_die( __( 'You are not allowed to do this!.', 'locale' ) );
}
```

_Any_ component of your theme on the front-end or your plugin on the back-end can be secured in this way. Then, you can assign this role to _just_ the people who are meant to manage advertisements (and can prevent anyone _lacking_ this specific role from doing anything they aren't supposed to).

## Code Securely

There are also coding best practices every WordPress developer should follow that protect the site _regardless_ of the capabilities granted to any specific user.

### 7. Sanitize All Input

Firstly, never trust any information provided by the user, be that user logged in or anonymous. Any input could be nefarious, and should be sanitized before any other code touches it:

[![Little Bobby Tables](http://imgs.xkcd.com/comics/exploits_of_a_mom.png)](https://xkcd.com/327/)

In a WordPress world, this means passing any `$_GET` or `$_POST` variables through some explicit filters:

```php
// Some basic type casting
$integer = absint( $_GET['number'] ); // Explicitly cast as an integer > 0
$boolean = (bool) $_POST['selected']; // Native PHP Boolean cast

$title = sanitize_title( $_POST['title'] );      // Sanitize a title input
$input = sanitize_textarea( $_POST['content'] ); // Sanitize a freeform text field

// When the content should contain HTML, pass it through "kses" to "strip evil scripts"
$content = wp_kses_post( $_POST['content'] );

// If you need to whitelist a specific array of tags, instead use wp_kses
$content = wp_kses( $_POST['input'], array(
    'a'      => array(
        'href'  => array(),
        'title' => array(),
    ),
    'br'     => array(),
    'em'     => array(),
    'strong' => array(),
) );
```

**Note:** While I explicitly mention "users" here, a user could be anything providing data into WordPress. This could be a human filling out a form, it could be a browser passing in a user agent string, or it could be a remote service delivering weather data for the site. A user is, for all intents and purposes, any source from which your data originates.

### 8. Escape All Output

Likewise, when printing output to the browser, you should treat your database as you would a user: don't trust anything it tells you. Instead, you should _escape_ any variable content before it's added to the HTML of the page.

For example, say a typical post would look like:

```php
<article data-id="<?php echo $post->id; ?>">
    <title><?php echo $post->title; ?></title>
    <section>
        <?php echo $post->content; ?>
        <a href="<?php echo $post->permalink; ?>">Read more...</a>
    </section>
</article>
```

Instead of merely trusting these properties of the `$post` object to be safe, we pass each through a purpose-built escaping function:

```php
<article data-id="<?php echo esc_attr( $post->id ); ?>">
    <title><?php echo esc_html( $post->title ); ?></title>
    <section>
        <?php echo esc_html( $post->content ); ?>
        <a href="<?php echo esc_url( $post->permalink ); ?>">Read more...</a>
    </section>
</article>
```

In detail, WordPress supports the following default functions for variable escaping:

- [`esc_attr()`](https://codex.wordpress.org/Function_Reference/esc_attr) for HTML attributes
- [`esc_html()`](https://codex.wordpress.org/Function_Reference/esc_html) for content for HTML
- [`esc_url()`](https://codex.wordpress.org/Function_Reference/esc_url) for URLs
- [`esc_js()`](https://codex.wordpress.org/Function_Reference/esc_js) for values passed to JavaScript
- [`esc_textarea()`](https://codex.wordpress.org/Function_Reference/esc_textarea) for values placed in a textarea
 
For the full list of sanitization functions available (including those used for the internationalization of UI strings) see the [WordPress Codex](https://codex.wordpress.org/Data_Validation).

### 9. Keep Everything Updated

When it comes to security updates, WordPress already does a fantastic job of keeping you protected. Since version 3.7, WordPress has shipped critical security patches _automatically_ and proactively updated installations to prevent anyone from being hacked.

In rare cases, this functionality extends to WordPress plugins and themes as well.

However, not all updates are automatic. New features are never forced on users; major updates to WordPress are opt-in only. In addition, WordPress can only update itself - keeping the stack upon which it relies updates is entirely in your hands.

Is PHP up to date? Has OpenSSL been patched? Are you running the latest and most stable version of Nginx? Is MySQL secure?

A chain is only as strong as its weakest link - even if WordPress is updated and secure, it's far from the only component running your website.

## Cultivate Professional Paranoia

The single most important thing you can do to protect and secure your WordPress site, however, is to merely think like an attacker. What are the weak points of your installation? What is the easiest way someone could breach your security?

Security professionals often talk of cultivating a ["professional paranoia."](http://ttmm.io/tech/professional-paranoia/) This means you look at your application and gauge the relative security or each and every component - and the risk each component would pose to your business if compromised.

As mentioned earlier, WordPress itself is a wildly popular target for attack. Even if you run the smallest site in the world, someone can _and will_ try to break your site. Whether or noy they're successful depends on what steps you take to protect your content.

Use strong passwords, and don't ever reuse them. Only browser your site securely, over networks you trust. Manage your user accounts with an iron fist and ensure that no one person has more access than they should.

As a developer, never trust content from your users (human or otherwise). Never trust your own database. Above all else, keep all of the components of your web stack up-to-date.

And always, _always_ assume that someone, somewhere will try to break the site you've invested time and energy to build. This viewpoint will help keep you on your toes and ensure you're always taking the steps required to keep WordPress running in the most secure manner possible.