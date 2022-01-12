---
layout: post
title:  "Simulate geolocation with Capybara and Headless Chrome"
date:   2022-01-11 23:02:32 +0000
categories: capybara
---
I recently added a "Locate me" button to [Film Chase](https://www.filmchase.com), which uses the Geolocation API (specifically [`getCurrentPosition`](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation/getCurrentPosition)) to get the current geolocation of a users device. Clicking the "Locate me" button triggers some Javascript like this

```javascript
getPosition() {
  navigator.geolocation.getCurrentPosition((position) => {
    this.filter(position.coords.latitude, position.coords.longitude);
  });
}
```

I was keen to add a feature specification to test this functionality

```ruby
it 'allows a user to share their current geolocation', :js do
  visit root_path

  click_button 'Locate me'

  expect(page).to have_text 'Location: Oxford'
end
```

This test won't work because thankfully headless Chrome isn't configured by default to share geolocation. In Chrome DevTools, a geolocation can be simulated using [geolocation override](https://developer.chrome.com/docs/devtools/device-mode/geolocation/). This is also possible within headless Chrome using [`Emulation.setGeolocationOverride`](https://chromedevtools.github.io/devtools-protocol/tot/Emulation/#method-setGeolocationOverride).

[Film Chase](https://www.filmchase.com) uses a pretty vanilla Rails 7 testing stack

```ruby
gem "capybara"
gem "selenium-webdriver"
gem "webdrivers"
```

Selenium helpfully provides a method to execute Chrome DevTools Protocol commands, succinctly named [`execute_cdp`](https://github.com/SeleniumHQ/selenium/blob/5db9c468557289608b0226a77f12f1b4dd511151/rb/lib/selenium/webdriver/common/driver_extensions/has_cdp.rb#L31) so within the context of a `js` enabled feature specification Capybara's driver can be accessed via `page.driver` and Selenium's driver via `page.driver.browser`. Meaning headless Chrome geolocation override can be enabled by appending the feature specification with

```ruby
page.driver.browser.execute_cdp(
  'Emulation.setGeolocationOverride',
  latitude: 51.7,
  longitude: -1.3,
  accuracy: 50
)
```

Attempting to run that test still won't work because we also need to grant the browser permission to access the geolocation, and we can achieve this with

```ruby
page.driver.browser.execute_cdp(
  'Browser.grantPermissions',
  origin: page.server_url,
  permissions: ['geolocation']
)
```

So putting this all together, we have

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

The test is now working so we can extract this behaviour into a helper method

```ruby
module HeadlessChromeHelpers
  def geolocation_override(latitude:, longitude:, accuracy: 100)
    grant_permissions('geolocation')
    set_geolocation_override(
      latitude: latitude,
      longitude: longitude,
      accuracy: accuracy
    )
  end

  private

  def grant_permissions(*permissions, origin: nil)
    origin ||= page.server_url

    page.driver.browser.execute_cdp(
      'Browser.grantPermissions',
      origin: origin,
      permissions: permissions
    )
  end

  def set_geolocation_override(coordinates)
    page.driver.browser.execute_cdp(
      'Emulation.setGeolocationOverride',
      **coordinates
    )
  end
end

RSpec.configure do |config|
  config.include HeadlessChromeHelpers
end
```

and now the feature specification can be refactored to be

```ruby
it 'allows a user to share their current geolocation', :js do
  geolocation_override(latitude: 51.7, longitude: -1.3)

  visit root_path

  click_button 'Locate me'

  expect(page).to have_text 'Location: Oxford'
end
```

I hope this is helpful!
