# Overview

In order to serve a web map, you need to serve the underlying map data in some
way. The most common way to do this is to pay a company like Google or Mapbox to
license their data. Of course, this costs money. Furthermore, you're tied to the
service provider. In the summer of 2018, [Google raised
prices][google-raise-prices] by 14x almost overnight. If you'd built your
platform on Google Maps, you're stuck.

[google-raise-prices]: https://gadgets.ndtv.com/apps/features/google-maps-apis-new-pricing-impact-1907242

In any case, I wouldn't use Google Maps because their platform is much less
friendly to developers. Mapbox's open source mapping apps are really easy to
build upon, and are at the forefront of map tech.

Mapbox is much friendlier, with a rather [generous free tier][mapbox-pricing].
Going with Mapbox services is not a bad option, but I chose against it for a few
reasons:

[mapbox-pricing]: https://www.mapbox.com/pricing/

- **Free project**. NST Guide is designed to be a free product, so I wanted to
  minimize current and future costs. By hosting my own map data, I can have
  clarity about expected costs with low variance month-to-month.
- **Infrequent updates okay**. One of the benefits of hosted maps like Mapbox is
  that they update from OpenStreetMap or other providers often. OpenStreetMap
  data is updated by contributors every day, so if you're interested in an area
  with a high rate of changes, you might be more interested in paying for their
  maps. For a trail map, constant updates aren't as important.
- **Don't need worldwide coverage**. My app is focused on the U.S. West Coast to
  start, and potentially eventually the whole US. I never expect to need
  coverage outside the continental US, so generating map data is much less
  complex than for the entire globe.
- **Learning experience**. A main goal with this project is to learn how map
  technology works, and by generating my own tiles I've been able to get a
  deeper understanding of vector tiles.
- **Low risk**. If I ever run into problems with my self-hosted map data, I can
  change a line of code, add a Mapbox API key, and be using their basemaps
  within an hour.
