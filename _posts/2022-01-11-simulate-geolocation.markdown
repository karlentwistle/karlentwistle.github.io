---
layout: post
title:  "Simulate geolocation with Capybara and Chrome headless"
date:   2022-01-11 23:02:32 +0000
categories: capybara
---
I recently added a "Locate me" button to [Film Chase](https://www.filmchase.com), which uses the Geolocation API (specifically [`getCurrentPosition`](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation/getCurrentPosition)) to get the current position of the users device.

Clicking the `Locate me` button triggers some Javascript like this:

```javascript
getPosition() {
  navigator.geolocation.getCurrentPosition((position) => {
    this.filter(position.coords.latitude, position.coords.longitude);
  });
}
```

I was keen to add a Capybara feature specification to test this functionality:

```ruby
it 'allows a user to share their current geolocation', :js do
  visit root_path

  click_button 'Locate me'

  expect(page).to have_text 'Location: Oxford'
end
```

This test won't work because headless Chrome isn't configured by default to share your geolocation, so we need to configure this behaviour.

Within Chrome DevTools, you can simulate a [geolocation override](https://developer.chrome.com/docs/devtools/device-mode/geolocation/), and this is also possible with Chrome headless using [`Emulation.setGeolocationOverride`](https://chromedevtools.github.io/devtools-protocol/tot/Emulation/#method-setGeolocationOverride). Selenium helpfully provides a method to execute Chrome DevTools Protocol commands, succinctly named [`execute_cdp`](https://github.com/SeleniumHQ/selenium/blob/5db9c468557289608b0226a77f12f1b4dd511151/rb/lib/selenium/webdriver/common/driver_extensions/has_cdp.rb#L31). Since I am using [Capybara](https://github.com/teamcapybara/capybara) the underlying Selenium driver is accessible in the feature specification context via `page.driver.browser`.

We can enable a headless Chrome geolocation override by appending the specification with:

```ruby
page.driver.browser.execute_cdp(
  'Emulation.setGeolocationOverride',
  latitude: 51.7,
  longitude: -1.3,
  accuracy: 50
)
```

Attempting to run that specification won't work yet because we also need to grant the browser permission to access the geolocation, and we can achieve this with:

```ruby
page.driver.browser.execute_cdp(
  'Browser.grantPermissions',
  origin: page.server_url,
  permissions: ['geolocation']
)
```

So putting this all together, we have:

```ruby
it 'allows a user to share their current geolocation', :js do
  page.driver.browser.execute_cdp(
    'Emulation.setGeolocationOverride',
    latitude: 51.7,
    longitude: -1.3,
    accuracy: 50
  )
  page.driver.browser.execute_cdp(
    'Browser.grantPermissions',
    origin: page.server_url,
    permissions: ['geolocation']
  )

  visit root_path

  click_button 'Locate me'

  expect(page).to have_text 'Location: Oxford'
end
```

The test is now working so we can extract this behaviour into a helper method:

```ruby
module GeolocationHelpers
  def geolocation_override(coordinates)
    page.driver.browser.execute_cdp(
      'Browser.grantPermissions',
      origin: page.server_url,
      permissions: ['geolocation']
    )

    coordinates.reverse_merge!(accuracy: 100)

    page.driver.browser.execute_cdp(
      'Emulation.setGeolocationOverride', **coordinates
    )
  end
end

RSpec.configure do |config|
  config.include GeolocationHelpers
end
```

and now the feature specification can be refactored to be:

```ruby
it 'allows a user to share their current geolocation', :js do
  geolocation_override(latitude: 51.7, longitude: -1.3)

  visit root_path

  click_button 'Locate me'

  expect(page).to have_text 'Location: Oxford'
end
```

Marvellous!
