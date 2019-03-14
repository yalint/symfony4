# Mime Type Validation

Unless the authors that can upload these files are super, super trusted, we need
some validation. At this moment, an author could upload literally *any* file type
to the system.

No problem: find the controller. Hmm, there's no form here. In
`ArticleAdminController`, we put the validation into the form. Then we could check
`$form->isValid()` and any errors rendered automatically on the form.

## Manually Validating

Because we're *not* inside a form, we need to validate directly... which is totally
fine! Add another argument: `ValidatorInterface $validator`. This is the service
that the form system uses internally for validation.

Then, before we do *anything* with that uploaded file, say
`$violations = $validator->validate()`. Pass this the object that you want to
validate. For us, it's just the `$uploadedFile` object itself. If we stopped here,
it would read any validation annotations off of that class and apply those rules.
But, because this is a core Symfony class, we can't open it up and add those. So,
pass a second argument: an the constraint to validate against.

Remember: there are two main constraints for uploads: the `Image` constraint that
we used before and the more generic `File` constraint, which we need here because
the user can upload more than just images. Say `new File()` - the one from the
`Validator` component.

This constraint has two main options - and the first is `maxSize`. Set it to `1k`
temporarily - just so we can see the error.

This `$violations`variable is *basically* an array of errors... except it's not
*actually* an array - it's an object that holds errors. To check if *anything*
failed validation, we can say if `$violations->count()` is greater than `0`. If
so, for now, let's just `dd($violations)` so we can see what it looks like.

Cool! Move over, select the Best Practices PDF - that's definitely more than 1kb,
and upload! Say hello to the `ConstraintViolationList`: a glorified array of
`ConstraintViolation` error objects. And there's the message: the file is too
large. If you want, you can customize that message by passing the `maxSizeMessage`
option... cause it *is* kind of a nerdy message.

## Displaying the Validation Errors

So, in theory, you can have multiple validation rules and multiple errors. To
keep things simple, let's show the first error if there is one. Use
`$violation = $violations[0]` to get it. The `ConstraintViolationList` class
implements `ArrayAccess`, which is why we can use this syntax. Oh, and let's help
out my editor by telling it that this is a `ConstraintViolation` object.

And now... hmm... how *should* we show this error to the user? This controller
will *eventually* turn into an AJAX, or API endpoint that communicates via JSON.
We'll do that soon when we make the whole reference upload system work via AJAX.
But because this is still a normal form submit, the easiest option is to put the
error into a flash message and display it on the next page. Say `$this->addFlash()`,
pass it an "error" type, and then `$violation->getMessage()`. Finish by stealing
the redirect code from the bottom to send us back to the edit page.

To *render* that flash message, open `templates/base.html.twig` and scroll down...
I'm looking for the flash message logic we added in our Symfony series. There
it is! We're rendering `success` messages, but we don't have anything to render
`error` messages. Copy this, paste, and loop over `error`. Make it look scary
with `alert-danger`.

Cool! Test it out - refresh! And... nice! It redirects and *there* is our error.

## Validating the Mime Types

This is great... but what we *really* want to do is control the *types* of files
that are uploaded. Change the max size to `5m` and add a `mimeTypes` option set
to an array. Let's see... what *do* we want to allow? Well, probably *any* image
is ok - so we can use `image/*` and definitely we should allow `application/pdf`.
But... what else? It's tricky - there are a lot of mime types out there. A nice
way to cheat is to press Shift+Shift and look for a core class called
`MimeTypeExtensionGuesser`.

This is a pretty neat class - it's what Symfony uses behind the scenes to "guess"
the correct file extension based on the mime type of a file. It's useful right *now*
because it has a *huge* list of mime types and their extensions. Check it out:
search for `'doc'`. There it is: `application/msword`. And if you keep digging
for other things like `docx` or `xls`, you can get a pretty good list of stuff
you might want to accept.

Close this file and go back to the option - I'll paste in a few mime types. This
covers a lot your standard "document" stuff. Oh, I forgot one! Add
`application/vnd.ms-excel`.

Let's try it out! Go back, select the Best Practices PDF, Upload and... no error!
Try it again - but with this `earth.zip` file - that's a zip of two of these photos.
Try it! Error! But *wow* is that a wordy error. You an customize this message with
the `mimeTypesMessage` option.

## Requiring the File

Ok: there's *one* last case we need to validate for. Hit enter on the URL to refresh
the form. Do *nothing* and hit upload. Ah!!! Whoops! Everything explodes inside
`UploaderHelper`... because there *is* no uploaded file!

Back in the controller, the second argument to `validate()` can accept an *array*
of validation constraints. Put the `new File` into an array. Then add:
`new NotBlank()` with a custom message: please select a file to upload.

Refresh one more time. The huge error is replaced by a *much* more pleasant validation
error.

Next: the author can *upload* a file reference... but it is literally impossible
for them to *download* it. How can we make these private files accessible, but
while still checking security first?