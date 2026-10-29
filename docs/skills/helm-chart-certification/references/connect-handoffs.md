# Red Hat Connect portal — human hand-offs

No API equivalent exists in the CI repo. Stop, hand over, wait.

> **HUMAN HAND-OFF — create the product listing.** Partner Connect → *Certified technology
> portal*. Create a listing by product category, complete the company profile, accept the legal
> agreements, add at least one component. Hand over: chart name, vendor name, vendor label.
> Wait for: the listing to exist before any component work.

> **HUMAN HAND-OFF — create and configure the Helm chart component.** Partner Connect →
> component → *Repository information* and *Component details* tabs. Set the container registry
> namespace, the Helm chart repository location, access instructions, and the public PGP key if
> signing. Hand over: the ASCII-armored public key and the GitHub usernames to authorize.
> Wait for: `redhat-mercury-bot` to land the `OWNERS` file at
> `charts/partners/<org>/<chart>/OWNERS`. Do not proceed to the PR until it exists.

> **HUMAN HAND-OFF — certify the referenced container images.** Every image in the chart needs
> its own certified, published container component before `images-are-certified` can pass. Hand
> over: the image list from the rendered templates. Wait for: *Published* status in the Red Hat
> Ecosystem Catalog; confirm with `pyxis-product-catalog`.

> **HUMAN HAND-OFF — publish.** Partner Connect → *My Work* → *Product Listings* → the listing →
> *Certification Projects* → *Attach Project*. Attach the chart project and the certified
> container project, both *Published*, then *Publish*. Hand over: the merged PR URL and the
> release tag. Wait for: the chart appearing in the Red Hat Ecosystem Catalog.
