---
layout: post
title:  "What is slowing down the startup of a rails app"
date:   2015-06-21 23:10:27
categories: rails
---

## Problem

I recently moved on to a previous rails app which was initialized by me and
another developer almost 8 months ago.
I can't remember when it began that the startup of this app takes almost a
minute in development mode.

It's been OK as rails has auto reload in development mode.
So I just left it alone.

However, things changed quickly while I started working on something that
requires change of initializers constantly and then leads to a server restart.

Suddenly that 1 minute startup time became unbearable and it's time to
correct it.

## Action

### Slow gem?
Loading gems is the first step of booting a rails app.

[Bumbler](https://github.com/nevir/Bumbler) is a tool to track the load progress
of a Bundler-based projects.

Install it and execute `bumbler` in rails root directory.

~~~
Slow requires:
    118.31  prawn
    154.96  twitter
    185.61  delayed_job_active_record
    212.49  grape
    216.75  compass
    218.28  devise
    291.53  bootstrap-sass
    292.53  cancancan
    314.34  pry
    332.67  capybara
    346.04  rails
    393.05  activeadmin
    583.57  fog
    669.51  mailboxer
~~~

None of these gems takes over a second to load. So the problem wasn't a slow gem.

### Slow initializer?!
The next step of booting a rails app is loading initializers.
Bumbler provides an easy command `bumbler --initializers` to see how slow your app's initializers are.

And this is the result I got.

~~~
Slow requires:
  44058.77  :set_routes_reloader_hook
~~~

That's it. A initializer takes 44 seconds to load which is definitly unnormal.

Then do a grep to locate the initilizer.

~~~
railties-4.1.10/lib/rails/application/finisher.rb-66-
railties-4.1.10/lib/rails/application/finisher.rb-67-      # Set routes reload after the finisher hook to ensure routes added in
railties-4.1.10/lib/rails/application/finisher.rb-68-      # the hook are taken into account.
railties-4.1.10/lib/rails/application/finisher.rb:69:      initializer :set_routes_reloader_hook do
railties-4.1.10/lib/rails/application/finisher.rb-70-        reloader = routes_reloader
railties-4.1.10/lib/rails/application/finisher.rb-71-        reloader.execute_if_updated
railties-4.1.10/lib/rails/application/finisher.rb-72-        self.reloaders << reloader
~~~

### What's next?
Next I try to use [Pry](https://github.com/pry/pry) and
[pry-byebug](https://github.com/deivid-rodriguez/pry-byebug)
to find out the exact slow function call.

The methodology is simple:
-> set a `break` point
-> use `next` to find the slow method in current frame
-> `step` into the slow method
-> set a `break` point
-> ...

First, in `config/environment.rb`, add `binding.pry` before
`Rails.application.initialize!`

~~~ ruby
    # Load the Rails application.
    require File.expand_path('../application', __FILE__)

    # Initialize the Rails application.
    binding.pry
    Rails.application.initialize!
~~~

Then execute `rails s`:

~~~
From: /Users/JJ/blackcitrus/secret/secret_rails/config/environment.rb @ line 6 :

    1: # Load the Rails application.
    2: require File.expand_path('../application', __FILE__)
    3:
    4: # Initialize the Rails application.
    5: binding.pry
 => 6: Rails.application.initialize!
~~~

Set a break point in `set_routes_reloader_hook`:

~~~
[1] pry(main)> break /Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/railties-4.1.10/lib/rails/application/finisher.rb:70

  Breakpoint 1: /Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/railties-4.1.10/lib/rails/application/finisher.rb @ 70 (Enabled)

    67:       # Set routes reload after the finisher hook to ensure routes added in
    68:       # the hook are taken into account.
    69:       initializer :set_routes_reloader_hook do
 => 70: reloader = routes_reloader
    71: reloader.execute_if_updated
    72: self.reloaders << reloader
    73: ActionDispatch::Reloader.to_prepare do
~~~

Execute `continue`

~~~
[2] pry(main)> continue

  Breakpoint 1. First hit

From: /Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/railties-4.1.10/lib/rails/application/finisher.rb @ line 71 :

    66:
    67:       # Set routes reload after the finisher hook to ensure routes added in
    68:       # the hook are taken into account.
    69:       initializer :set_routes_reloader_hook do
 => 70:         reloader = routes_reloader
    71:         reloader.execute_if_updated
    72:         self.reloaders << reloader
    73:         ActionDispatch::Reloader.to_prepare do
    74:           # We configure #execute rather than #execute_if_updated because if
    75:           # autoloaded constants are cleared we need to reload routes also in
    76:           # case any was used there, as in
~~~

Execute `next`:

~~~
[2] pry(#<Secrect::Application>)> next

From: /Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/railties-4.1.10/lib/rails/application/finisher.rb @ line 71 :

    66:
    67:       # Set routes reload after the finisher hook to ensure routes added in
    68:       # the hook are taken into account.
    69:       initializer :set_routes_reloader_hook do
    70:         reloader = routes_reloader
 => 71:         reloader.execute_if_updated
    72:         self.reloaders << reloader
    73:         ActionDispatch::Reloader.to_prepare do
    74:           # We configure #execute rather than #execute_if_updated because if
    75:           # autoloaded constants are cleared we need to reload routes also in
    76:           # case any was used there, as in
~~~

The execution quickly steps over to the next line.

Execute `next`:

~~~
[3] pry(#<Secrect::Application>)> next
~~~

This time it takes around 1 minute to step over to the next line.
So `71: reloader.execute_if_updated` is the line we are looking for.

Stop current execution, restart server and set a break point at this line.

Repeat what I have just done...

However quickly, after ten minutes, I realized rails is a very complex system and
what I am doing now may goes forever.

### The right way - profiling and benchmark
What I need at this moment is [ruby-prof](https://github.com/ruby-prof/ruby-prof).

`bundle open` railties, in `lib/rails/application/finisher.rb`, add code:

~~~ruby
  initializer :set_routes_reloader_hook do
    reloader = routes_reloader
    puts 'start prof'
    RubyProf.start
    reloader.execute_if_updated
    results = RubyProf.stop
    puts 'end prof'
    File.open "#{Rails.root}/tmp/rubyprof/stack_#{Time.now.to_s(:db)}.html", 'w' do |file|
      RubyProf::CallStackPrinter.new(results).print(file)
    end
    ...
  end
~~~

Execute `rails s`, then I got a [call stack tree graph]({{ site.url }}/assets/rubyprof_stack.html) in html format.

From the graph I can see there are a lot of calls related to
[Psych](http://ruby-doc.org/stdlib-2.2.1/libdoc/psych/rdoc/Psych.html), which is
a YAML parser. And it all starts from 'I18n::Backend::Base#load_file'.

Click on the method name, it will open the file in text editor.

To know what files are loaded and how long it takes to load each file, add
benchmark code:

~~~ruby
  def load_file(filename)
    puts "#{filename}"
    result = Benchmark.ms do
      type = File.extname(filename).tr('.', '').downcase
      raise UnknownFileType.new(type, filename) unless respond_to?(:"load_#{type}", true)
      data = send(:"load_#{type}", filename)
      unless data.is_a?(Hash)
        raise InvalidLocaleData.new(filename, 'expects it to return a hash, but does not')
      end
      data.each { |locale, d| store_translations(locale, d || {}) }
    end
    puts result
  end
~~~

Execute `rails s`:

~~~
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/activesupport-4.1.10/lib/active_support/locale/en.yml
741.9292960548773
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/activemodel-4.1.10/lib/active_model/locale/en.yml
212.11264096200466
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/activerecord-4.1.10/lib/active_record/locale/en.yml
71.97770604398102
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/actionview-4.1.10/lib/action_view/locale/en.yml
394.5407139835879
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/grape-0.11.0/lib/grape/locale/en.yml
354.5366879552603
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/de-AT.yml
16643.352392013185
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/de-CH.yml
309.91789093241096
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/de.yml
21681.582821998745
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/en-au-ocker.yml
1337.0081200264394
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/en-AU.yml
2913.1630359916016
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/en-BORK.yml
507.7939030015841
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/en-CA.yml
429.1196570266038
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/en-GB.yml
584.9791240179911
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/en-IND.yml
4618.128058034927
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/en-NEP.yml
967.6744639873505
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/en-US.yml
3325.105574913323
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/en.yml
32816.76925998181
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/es.yml
9538.302210974507
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/fa.yml
4400.56179696694
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/fr.yml
10188.019018038176
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/it.yml
6565.096735022962
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/ja.yml
885.8637060038745
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/ko.yml
1651.7194519983605
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/nb-NO.yml
2635.0025959545746
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/nl.yml
7878.697662032209
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/pl.yml
19974.775120965205
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/pt-BR.yml
3545.6651960266754
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/ru.yml
6768.859902047552
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/sk.yml
23113.826932036318
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/sv.yml
3274.18546192348
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/vi.yml
2200.5813339492306
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/faker-1.4.3/lib/locales/zh-CN.yml
2095.091123948805
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/ransack-1.6.6/lib/ransack/locale/cs.yml
656.753116985783
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/ransack-1.6.6/lib/ransack/locale/en.yml
741.558330017142
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/ransack-1.6.6/lib/ransack/locale/es.yml
623.7398190423846
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/ransack-1.6.6/lib/ransack/locale/fr.yml
600.3050709841773
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/ransack-1.6.6/lib/ransack/locale/hu.yml
629.7791349934414
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/ransack-1.6.6/lib/ransack/locale/nl.yml
643.7799989944324
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/ransack-1.6.6/lib/ransack/locale/ro.yml
665.7998530426994
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/ransack-1.6.6/lib/ransack/locale/zh.yml
608.7534320540726
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/responders-1.1.2/lib/responders/locales/en.yml
67.63134803622961
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/carrierwave-0.10.0/lib/carrierwave/locale/en.yml
88.2780869724229
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/paperclip-4.2.0/lib/paperclip/locales/de.yml
125.07579394150525
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/paperclip-4.2.0/lib/paperclip/locales/en.yml
132.61232199147344
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/paperclip-4.2.0/lib/paperclip/locales/es.yml
123.02663701120764
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/will_paginate-3.0.7/lib/will_paginate/locale/en.yml
119.39483799505979
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/devise-3.3.0/config/locales/en.yml
477.60191606357694
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/mailboxer-0.12.3/config/locales/en.yml
44.32142700534314
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/kaminari-0.16.3/config/locales/kaminari.yml
140.27521200478077
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/formtastic_i18n-0.4.1/config/locales/ca.yml
84.62769794277847
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/formtastic_i18n-0.4.1/config/locales/de.yml
83.09465704951435
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/formtastic_i18n-0.4.1/config/locales/es.yml
75.08417509961873
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/formtastic_i18n-0.4.1/config/locales/it.yml
90.2807789389044
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/formtastic_i18n-0.4.1/config/locales/pl.yml
37.82614099327475
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/formtastic_i18n-0.4.1/config/locales/pt-BR.yml
85.14734904747456
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/formtastic_i18n-0.4.1/config/locales/pt.yml
85.03939106594771
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/formtastic_i18n-0.4.1/config/locales/tr.yml
79.45211604237556
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/gems/formtastic_i18n-0.4.1/config/locales/zh-CN.yml
85.1048119366169
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/ar.yml
1145.8351940382272
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/bg.yml
940.2479939162731
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/bs.yml
1046.421576058492
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/ca.yml
911.3963799318299
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/cs.yml
918.0804250063375
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/da.yml
879.6392700169235
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/de-CH.yml
897.9943880112842
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/de.yml
1122.114620055072
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/el.yml
1136.2147459294647
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/en-GB.yml
798.7757149385288
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/en.yml
1092.8969079395756
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/es-MX.yml
839.3011289881542
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/es.yml
1048.781214049086
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/fa.yml
1006.2566479900852
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/fi.yml
935.244056978263
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/fr.yml
974.6984719531611
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/he.yml
826.7514749895781
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/hr.yml
1040.5551790026948
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/hu.yml
868.6606859555468
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/it.yml
983.0245879711583
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/ja.yml
1101.2658659601584
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/ko.yml
803.4766339696944
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/lt.yml
1047.3589000757784
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/lv.yml
790.2443470666185
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/nb.yml
973.8541450351477
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/nl.yml
1079.586865962483
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/pl.yml
780.5351580027491
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/pt-BR.yml
1038.037788006477
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/pt-PT.yml
739.1904569230974
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/ro.yml
768.3917579706758
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/ru.yml
1119.3809249671176
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/sv-SE.yml
732.1475809440017
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/tr.yml
782.102567027323
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/uk.yml
1110.3377710096538
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/vi.yml
769.5566279580817
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/zh-CN.yml
898.6905079800636
/Users/JJ/.rbenv/versions/2.2.1/lib/ruby/gems/2.2.0/bundler/gems/activeadmin-0b4b22871fd3/config/locales/zh-TW.yml
1038.202345953323
/Users/JJ/blackcitrus/secret/secret_rails/config/locales/devise.en.yml
440.7359679462388
/Users/JJ/blackcitrus/secret/secret_rails/config/locales/en.yml
12.340541929006577
~~~

Finally I found the culprit - [faker](https://github.com/stympy/faker).

## Conclusion
It turns out that `gem faker` was added to `group :developmen, :test`.

Moving it to `group :test` solved the problem.

P.S. A alternative to faker - [ffaker](https://github.com/EmmanuelOga/ffaker).
