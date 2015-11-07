# Google DFP Ad Targeting and Joomla

## Using the URL Path
One approach to targeting ads dynamically is to parse the URL and use part of it to define a DFP target for a given page.

For example, we can parse everything to the right of the last slash in `http://domain.com/accommodations` and dynamically set it to the value of the DFP 'setTargeting' method, which would look like: `googletag.pubads().setTargeting("category","accommodations");`

In DFP, any campaign running for domain.com that is targeted to `accommodations`, will run only on this page.

## Using Joomla Tags
Another approach would be to populate the value to DFP's `setTargeting()` method to a keyword tag.  This approach allows an ad campaign to be target to any article with that tag, regardless of its URL. This approach would not be appropriate when a campaign needs to run on non-article pages, like a Category Blog, that displays teasers of many articles at once.
