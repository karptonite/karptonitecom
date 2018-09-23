---
title: "Rehashing Password Hashes"
date: 2014-05-11T15:07:31-04:00
categories:
    - php
---

Every time a prominent website has their user database stolen, the question of
how the passwords were stored comes up. Even properly hashed password tables
are vulnerable to some attacks, but the better the hashing algorithm, the less
the risk to the users.


Tools have come a long way. In the PHP world, we now have the `password_hash()`
function [built into PHP
5.5](http://docs.php.net/manual/en/function.password-hash.php). However, many
of us work on websites which have a table of old passwords hashed by older,
less secure tools or by some home-brewed hashing system. When I read about
methods of upgrading old systems, the recommendation I sometimes see is that
the best you can do to correct this situation is have a system that validates
passwords when users log in using the old system, then [silently
rehashes](https://github.com/jeremykendall/password-validator#upgrading-legacy-passwords)
them with the new improved algorithm.  Unfortunately, this leaves infrequent
users, or users who created accounts and never logged in again, more
vulnerable, should your database be compromised.  But you can't create the
correct new hash without the original password,
[right](http://blog.stidges.com/post/upgrading-legacy-passwords-with-laravel)?

This is absolutely correct; luckily, there is another alternative: you can
rehash the old _hashes_.  Here is how it works:

Say your legacy systems works roughly as follows:

~~~php
public function savePassword(User $user, $password)
{
   $salt = $this->getRandomSalt();
   $hash = $this->hashPassword($password, $salt);
   $user->setPasswordHash($hash);
   $user->setPasswordSalt($salt);

public function checkUserPassword(User $user, $password)
{
   return $this->checkPassword($password, $user->getSalt(), $user->getHash())
}

protected function checkPassword($password, $salt, $hash)
{
   return $hash == $this->hashPassword($password, $salt);
}
~~~

When you want to upgrade to password_hash, run something resembling the following on
each user.
~~~php
public function convertUserPassword(User $user)
{
   $user->setPasswordVersion('legacy');
   $user->setNewPasswordHash(password_hash($user->getHash()));
}
~~~

Once the passwords are converted, you can change your password checking code:

~~~
public function checkUserPassword(User $user, $password)
{
   if($user->getPasswordVersion() =='legacy')
   {
      // using our old hashPassword function and our old salt
      $oldStyleHash = $this->hashPassword($password, $user->getSalt());
      return password_verify($oldStyleHash, $user->getNewPasswordHash());
      // if you want, now might be a good time to hash the actual password,
      // and upgrade the user's password version.
   }
   // else, use password_verify() as normal
}
~~~
Notice what we've done here: We don't have the user's password, but we do have a hash
that we know can be generated from the password and the proper salt. We've treated that
hash as the _password_ for the new, improved hashing system. We know that when the user
does type in their password, we will be able to regenerate that hash because we
still have the salt.[^1]

Now for the important part: you can delete all of the the old password hashes
(but not the salts) from the database and any backups. All users are now
protected by the new password hashing algorithm, even if they never log in
again.

Note that in practice, some legacy systems will have saved the salt and the
hash as part of the same string, but these should be separable. The important
thing is to keep the salt but discard the old hash.

Once the original password hashes are deleted, ALL of your users should benefit from
the improved hashing algorithm, not just those who log in again.

Last minute thought: This functionality could be built into the PHP password
hashing API directly for hashes originally created by the API.  A
`password_rehash()` function would take a password hash created with a
now-deprecated algorithm, rehash the hash as described in this post via the new
algorithm, and store both the old and new salt (and algorithm codes) in the new
password hash string, such that `password_verify()` could verify it.

[^1]: The first time a user logs in under the new system, you can hash the plain text password with `password_hash()`, and upgrade the password version accordingly. That way, if you ever drop support for the old password system entirely, anyone who's logged in since then will have their password saved in the new format. However, it is not necessary to wait for them to log in to delete the original hash.

