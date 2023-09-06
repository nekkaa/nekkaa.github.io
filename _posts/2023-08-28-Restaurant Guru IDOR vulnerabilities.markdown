---
title: Restaurant Guru IDOR vulnerabilities
layout: post
---

![restaurant-guru](/assets/restaurant-guru.png){:style="display:block; margin-left:auto; margin-right:auto"}

# Creating a comment
Restaurant Guru allows anyone to write a comment on the page of their favorite restaurant, either if the user is logged in or not, so thats what I did, I hooked up burp suite and wrote a comment.

![comment-add](/assets/comment-add.png)

This revealed the `/ajax/comment_add` endpoint which takes the form filled out by the user and creates a comment. Among all of the obvious fields that you would expect a comment form to contain, theres a few interesting ones that allow us to do many cool things with this endpoint, one of them being the `cid` field.

# Editing a comment
After posting my comment I realized that just writing `great!` for a such good restaurant wasn't enough, I needed to show some love to it! so I decided to edit it.

![comment-edit](/assets/comment-edit.png)

As we can see the same `/ajax/comment_add` endpoint is being used and the comment_id field, or `cid` for short, is responsibile for identifying the comment we want to edit, this field was set to `0` when we were creating the comment.

# Arbitrary comment overwrite
Once my comment was edited I was ready to see what the other people had to say about this restaurant, the comments were pretty good as expected but there were a few that looked a little fake and hateful.

I was a little disappointed in seeing that so I tried to figure out a way to edit these comments, the first thing I tried was setting the `cid` field of the form to comment id of the hateful comment. Finding the comment_id was pretty simple as we can just scrape it from the page.

![hateful-comment](/assets/hateful-comment.png)

The `cid` field should be set to `319242`, let's send a request to the `/ajax/comment_add` endpoint and see what happens.

![edited-hateful-comment](/assets/edited-hateful-comment.png)

Looks like it worked, the negative comment turned into a positive one, proving that no authorization checks have been implemented, anyone can overwrite anyones comment just by specifying the `cid` field.

# Arbitary comment deletion
After seeing what happened with the `/ajax/comment_add` endpoint, I started looking around for other endpoints that have similar issues. The first endpoint I wanted to take a look at is the one used for deleting comments.

![comment-delete](/assets/comment-delete.png)

This revealed the `/ajax/comment_delete` endpoint which takes the `id` of the comment, an `object_id` identifying  the restaurant and an `object_type`, we can easily get all of these just by scraping the page.

![restaurant-id](/assets/restaurant-id.png)

Now that we know what parameters the endpoint takes, let's try and delete someone elses comment. To try this out I wrote a comment from another browser, so that a new account is being used and I avoid deleting someones actual comment.

![nekka-other-comment](/assets/nekka-other-comment.png)

The id of the comment is `1036556`, the object_id and object_type are the same as before because I'm writing a comment for the same restaurant.
Let's send a request to the `/ajax/comment_delete` endpoint using the id of the other comment and see what happens.

![comment-deleted](/assets/comment-deleted.png)

Looks promising, refreshing the page indeed proves us that the comment was deleted, once again no authorization checks have been implemented allowing anyone to delete anyones comment.

# Arbitary image binding
The next thing I was curious about is how images are bound to comments, so I went ahead and added an image to the comment I initially posted.

![image-binding](/assets/image-binding.png)

As we can see the `/ajax/comment_upload_image` endpoint is being used, it takes a form with some basic information about the image, the image itself and most importantly a `comment_id` identifying the comment to which the image should be bound to.

![post-image-binding](/assets/image-binding-edit.png)

After this request has gone through a new request is made to the `/ajax/comment_add` endpoint to make sure any other change in the comment is saved.

We've already seen that the `/ajax/comment_add` endpoint allows anyone to overwrite anyones comment but let's check if the `/ajax/comment_upload_image` endpoint allows anyone to bind an image to any comment, to check this out I posted a comment using another account.

![bind-image-comment](/assets/bind-image-comment.png)

The `comment_id` is `1036608`, let's send a request to the `/ajax/comment_upload_image` endpoint.

![image-binding-outcome](/assets/image-binding-outcome.png)

The request went through and the comment now has an image, turns out that anyone can bind images to any comment.

# Arbitrary image deletion
After uploading the images I figured I'd like to see how they are deleted as well so I went ahead and deleted the image I posted before.

![image-deletion](/assets/image-deletion.png)

Seems like the `/ajax/comment_add` endpoint is also used to delete pictures in a comment, meaning that if you want to remove an image in a comment you need to overwrite the whole comment and specify the id of the picture to delete in the `img_ids_for_deletion[]` field. 

# Arbirary comment stars overwrite
Another cool endpoint I found reading the javascript code in the `comments.js` file is the `/ajax/comment_vote_change` endpoint, which allows us to set the stars of a comment.

![vote-change-js](/assets/vote-change-js.png)

The endpoint takes the number of stars in the `star` field, an `object_id` identifying the restaurant and `comment_id` identifying the comment for which to set the stars.

![vote-change-before](/assets/stars-before-1.png)

![vote-change-after](/assets/stars-after-1.png)

This is pretty useful if you are just trying to change stars of a comment without overwriting the whole thing, however if you are changing the stars of a comment you probably want to change the message of the comment as theres no point in having 5 stars but a negative message.
