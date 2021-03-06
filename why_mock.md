## Why Use Mock

   * Less interaction with e.g database
   * No need to unnecessarily expose state
   * Less duplicate coverage
   * Faster tests
   * Reduced coupling
   * Enhanced TDD


## Type of Mock

   * Dummy Object
   * Fake Object
   * Stubs
   * Mocks
  
### Dummy Object	  
	 
    Dummy objects are passed around but never actually used. Usually they are just used to fill parameter lists.

### Fake Object
	
	Fake objects actually have working implementations, but usually take some shortcut which makes them not suitable for production (an in memory database is a good example).

### Stubs
	
	Stubs provide canned answers to calls made during the test, usually not responding at all to anything outside what's programmed in for the test. Stubs may also record information about calls, such as an email gateway stub that remembers the messages it 'sent', or maybe only how many messages it 'sent'.

### Mocks
	
	Mocks are objects pre-programmed with expectations which form a specification of the calls they are expected to receive.


## Mock Libraries

   * [Mocha](https://github.com/freerange/mocha)
   * [RSpec Mocks](http://rubydoc.info/gems/rspec-mocks/frames)
   * [Flexmock](https://github.com/jimweirich/flexmock)
   * [Mock Time](https://github.com/travisjeffery/timecop)
   * [Fake File System](https://github.com/defunkt/fakefs)
   * [Fake Web Request](https://github.com/chrisk/fakeweb)
   * [Fake Redis](https://github.com/guilleiguaran/fakeredis)
   * [Fake S3](https://github.com/jubos/fake-s3)
   * [Fake Request like VCR](https://github.com/vcr/vcr)
   
   
   
## Further Reading  

   * [Mock Object](http://www.mockobjects.com/)
   * [Test Double](http://xunitpatterns.com/Test%20Double.html)
   * [An Introduction To Mock Objects In Ruby](http://jamesmead.org/talks/2007-07-09-introduction-to-mock-objects-in-ruby-at-lrug/)
   * [Mocks Aren't Stubs](http://martinfowler.com/articles/mocksArentStubs.html )
