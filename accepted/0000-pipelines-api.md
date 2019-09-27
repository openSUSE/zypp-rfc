- Feature Name: Zypper functional API with async support
- Start Date: 19-06-2019
- RFC PR: (leave this empty)

# Summary
[summary]: #summary

This proposes the addition of a asynchronous functional style framework to libzypp, to make it possible to build so called pipelines that can decrease the amount of code to be written and thus make it easier to maintain the codebase. 
Also by choosing a functional style we can avoid unwanted side effects. 

# Motivation
[motivation]: #motivation

When looking through the zypp codebase it is obvious that most of the operations that happen in zypper is mapping input to output data, for example mapping a list of requested packages to the list of installable packages, from the list of installable packages to the list of packages to download etc. 
This is a perfect use case for functional style programming. 

In the new zypper API we will have support for asynchronous code as well to take better advantage of system ressources, 
while using signals and slots to handle async is a proven and realiable method, using it everywhere in our code will lead to code that is harder to read, because code that belongs together is cluttered over several callback functions and it needs many state tracking variables.

# Detailed design
[design]: #detailed-design

A usual code example with signals and slots would look like (pseudocode):

```cpp
class PackageInstaller {

 public:
    //download and install a given set of packages
    void installPackages ( std::vector<std::string> packages ) {

        if ( packages.isEmpty() ) {
            setFinished(); //nothing to do 
            return;
        }

        for ( const std::string &pck : packages ) {
            PackageInfo pckInfo = Pool::instance().findPackage(pck)
            if !( pckInfo ){
                std::cout << "Package " << pck << " not found" << std::endl;
                continue;
            }
            if ( fileExists( pckInfo.cacheDir()+"/"+pckInfo.fileName() ) ) {
                installFileFromCache(pckInfo);
                continue;
            }
                
            Download::Ptr dl = Downloader::instance().downloadFile( pckInfo.url(), Config::instance().tmpDir );
            dl->sigFinished.connect( this, &PackageInstaller::packageDownloaded );
            _downloads.push_back( make_pair(dl, pckInfo) );
        }

        if ( _download.empty() && _runningInstalls.empty() ) {
            setFinished(); // nothing to do
        }
    } 

    SignalProxy<void()> sigFinished() { return _downloadingFinishedSignal; }
    any error () const { return _error; }

  private:
    void packageDownloaded (  Download &dl  ) {
        auto it = std::find_if (_downloads.begin(), _downloads.end(), [&dl]( const auto &pair ){ return pair.first().get() == &dl; });

        auto finished = *it;
        _downloads.erase(it); // remove from vector

        auto &dlPtr = finished.first();
        auto &dlSpec = finished.second();

        if ( dlPtr->hasError() ) {
            //package download failed, cancel
            setError( PackageDownloadFailedError( dlSpec ) );
            return;
        }

        if ( !checksumValid ( dlPtr->filePath(), dlSpec.checksum()  ) )  {
            //package checksum did not match, cancel
            setError( ChecksumMismatchError( dlPtr->filePath() ) );
            return;
        }

        if ( !moveFile ( dlPtr->filePath(), dlSpec.cacheDir() ) ) {
            setError( FileMoveError( dlPtr->filePath() ) );
            return;   
        }

        installFileFromCache(dlSpec);

        if ( _download.empty() && _runningInstalls.empty() ) {
            setFinished(); // nothing to do
        }
    }

    void installFileFromCache ( PackageInfo spec ) {
        Process::Ptr proc = Process::run( std::string("rpm -i ")+spec.cacheDir()+"/"+spec.fileName() );
        proc->sigFinished().connect( sigc::mem_fun(this, &PackageInstaller::installFinished ) );
    }

    void installFinished ( Process &proc ) {
        auto it = std::find_if (_runningInstalls.begin(), _runningInstalls.end(), [&proc]( const auto &p ){ return p.first().get() == &proc; });    

        auto finished = *it;
            
        auto &procPtr = finished.first();
        auto &procSpec = finished.second();
    
        if ( proc->exitCode() != 0 )  {
            setError( InstalledFailedError() );
            return;
        }
        
        if ( _download.empty() && _runningInstalls.empty() ) {
            setFinished(); // done
        }        
    }

    void setFinished () {
        _downloadingFinishedSignal.emit();
    }
    
    template<typename T>
    void setError ( T &&err ) {
        _err = std::move(err);

        //cancel all running tasks
        _downloads.clear();
        _runningInstalls.clear();

        setFinished();
    }

private:
    std::vector<std::pair<Download::Ptr,PackageInfo>> _downloads;
    std::vector<std::pair<Process::Ptr, PackageInfo>> _runningInstalls;
    sigc::signal<void()> _downloadingFinishedSignal;
    any _error;
};

```
The above example is small but already needs lots of explicit state handling and is
cluttered over several class member functions that make the code hard to read, especially if the class grows bigger and the functions
are not completely visible on one page.

With a functional approach we can do better, to better understand the code this are some of the
special callbacks used:

  * lift -> a helper function that allows us to remember values of type A by "lifing" or wrapping a function accepting type B and returning R to a function 
    accepting pair<B,A> and returning pair<R,A>
  * transform -> this is commonly known as "map" in other languages, however transform is the term used in the std library. A function that maps a set
    of values e.g. vector<A> into a set of values e.g vector<B> using a callback "f(A) -> B"
  * filter -> forward only values from the input set that pass the given predicate
  * mbind -> This function accepts a type of expected<A> and and a callback "f(A)->expected<B>", only executing the callback if the expected type does not contain
    a error, otherwise the error is directly forwarded.
  * waitWhile -> A function taking a set of AsyncResult instances, waiting for them to get ready. If a result gets ready the given callback is called, depending on its 
    return value the loop is cancelled or continues


```cpp
AsyncJobPtr< expected<void> > installPackages ( std::vector<std::string> packages ) {

    if ( packages.isEmpty() ) {
        return make_ready_result( expected<void>::error( ArgumentEmptyError() ) );
    }

    return = std::move(packages) 
        //search packages
        | transform ( []( std::string &&pck ) {
            PackageInfo pckInfo = Pool::instance().findPackage(pck)
            if !( pckInfo ){
                std::cout << "Package " << pck << " not found" << std::endl;
                return expected<PackageInfo>::error( PackageNotFoundError() );
            }
            return expected<PackageInfo>::success( std::move(pckInfo) );
          })
        //filter out all packages we did not find
        | filter( []( auto && expectedInfo ) {  return !expectedInfo.isError(); } ) 
        //at this point we want to download the files, or take them from cache
        | transform ( mbind( []( auto &&pckInfo ) -> AsyncJobPtr< expected<PackageInfo> > {

            if( fileExists( pckInfo.cacheDir()+"/"+pckInfo.fileName() ) )
                return make_ready_result(expected<PackageInfo>::success( std::move(pckInfo) )); //make a async result that is finished right away
               
            return std::move(pckInfo)
            //download the file
            | [] ( auto &&pckInfo ) { return make_pair( Downloader::instance().downloadFile( pckInfo.url(), Config::instance().tmpDir ), pckInfo );  }
            //wait for download to finish
            | lift( await<Downloader::Ptr>( &Downloader::sigFinished ) )
            | []( auto &&dlPair ) { 
                if( dl.first->hasError() ) return expected< pair<string,PackageInfo> >::error( PackageDownloadFailedError( dlPair.second ) );
                return expected<string, PackageInfo>::success( make_pair( dl.first->filePath(), dlPair.second ));
            } 
            | mbind( []( pair<string,PackageInfo> &&dloadedFile ){
                 checksumValid ( dloadedFile.first, dloadedFile.second.checksum())
                    ? return expected<string, PackageInfo>::success( dloadedFile ) 
                    : return expected<string, PackageInfo>::error( ChecksumMismatchError() );
            })
            | mbind( []( pair<string,PackageInfo> &&dloadedValidFile ){
                 if ( !moveFile ( dloadedValidFile.first, dloadedValidFile.second.cacheDir() ) ) {
                    return expected<PackageInfo>::error( FileMoveError( dloadedValidFile.first ) );
                 }
                 return expected<PackageInfo>::success( dloadedValidFile.second );
            })
            | mbind( []( PackageInfo &&spec ) {
                return make_pair( Process::run( std::string("rpm -i ")+spec.cacheDir()+"/"+spec.fileName() ), spec );
            })
            | lift( await<Process::Ptr>( &Process::sigFinished ) )
            | []( auto &&procPair ) {
                procPair.first->exitCode() == 0
                ? return expected<PackageInfo>::success( procPair.second )
                : return expected<PackageInfo>::error( InstalledFailedError() );
            }
            ;
            
          }))
        //wait for all the results, stopping if one errors out, cancelling the rest
        | waitWhile<zyppng::expected<DlRequest>>( []( auto &&res ){ return !res.hasError(); }  );
        | []( std::vector<expected<PackageInfo>> &&results ){
            auto errIt = std::find_if( results.begin(), results.end(), []( const auto &res ){ return res.hasError(); });
            if( errIt ) 
                return expected<void>::error( errIt->error() );
            return expected<void>::success();
        };
}
```

The resulting code is 48 lines shorter, has no need for variables to track all the running downloads or other async tasks. All ressources
are owned by the pipeline, freeing up the pipeline will also cancel all running async tasks inside. 

The basic idea here is to make it possible to build so called pipelines, bascially something where a input is put in on top ( set of packages ), the data goes through a defined set of transformations and a result comes out at the end ( success/failure ). 
There are several libraries available that support this for synchronous pipelines:

  * Boost Ranges: https://www.boost.org/doc/libs/1_71_0/libs/range/doc/html/index.html
  * RangesV3: https://github.com/ericniebler/range-v3
  * C++20 Ranges

However, those libraries do not support async pipelines. Wether we can enable them to work together with a async pipeline implementation needs to be tested during implementation,
usually they work by providing lazy evaluation by passing iterators instead of values through the pipeline, which implies that the initial set/vector/map/value needs
to be valid throughout the pipeline, something that is not given with a pipeline that does not own the input value.

In order to make this work, a basic building block is required that supports the possibility of having a asynchronous task:

```cpp
template<typename Res>
struct AsyncJob
{
    void setReady ( Res && val );

    template<typename T>
    void onReady ( T &&callback );

    bool isReady () const;
    Res &get ();
};

```

This is the base for all asynchronous operations that can be used in a pipeline. 
Implementing a async operation is as simple as:

```cpp
//a async operation returning a int
struct MyAsyncOp : public AsyncJob<int>
{
    void operator()( int &&inputValue ) {
        //wait for inputValue amount of seconds, then set the AsyncOp as ready
        EventLoop::instance().delayCall( inputValue, [this, inputValue]() mutable {
            //calling setReady is like calling return in a normal function, no more instructions are allowed after it
            setReady( std::move(inputValue) );
        });
    }
}
```

Also a way to connect several async and sync functions into a pipeline is required, this can be done with a set of binding functions.
For the sake of readability the proposal is to overload the pipe operator:

```cpp

//most simple version, sync result to sync function
Res operator| ( SyncResult &&in, SyncCallbackFunction &&callback );

// sync input , to async function
AsyncRes operator| ( SyncResult &&in, std::unique_ptr<AsyncCallback> &&callback );

// async input , to sync function
AsyncRes operator| ( std::unique_ptr<AsyncRes> &&in, SyncCallbackFunction &&callback );

// async input , to async function
AsyncRes operator| ( std::unique_ptr<AsyncRes> &&in, std::unique_ptr<AsyncCallback> &&in &&callback );

```

The last building block is a type that glues everything together and connects one operation to the next one, automatically
forwarding results when they are ready. 

```cpp
    template <typename Prev, typename AOp >
    struct AsyncResult<Prev,AOp> : public zyppng::AsyncOp< typename AOp::value_type > {

      AsyncResult ( std::unique_ptr<Prev> && prevTask, std::unique_ptr<AOp> &&cb )
        : _prevTask( std::move(prevTask) )
        , _myTask( std::move(cb) ) {
        connect();
      }

      AsyncResult ( const AsyncResult &other ) = delete;
      AsyncResult& operator= ( const AsyncResult &other ) = delete;

      AsyncResult ( AsyncResult &&other ) = delete;
      AsyncResult& operator= ( AsyncResult &&other ) = delete;

      virtual ~AsyncResult() {}

      void connect () {
        //not using a lambda here on purpose, binding this into a lambda that is stored in the _prev
        //object causes segfaults on gcc when the lambda is cleaned up with the _prev objects signal instance
        _prevTask->onReady( std::bind( &AsyncResult::readyWasCalled, this, std::placeholders::_1) );
        _myTask->onReady( [this] ( typename AOp::value_type && res ){
          this->setReady( std::move( res ) );
        });
      }

    private:
      void readyWasCalled ( typename Prev::value_type && res ) {
        //we need to force the passed argument into our stack, otherwise we
        //run into memory issues if the argument is moved out of the _prevTask object
        typename Prev::value_type resStore = std::move(res);

        //clean up ressources that are not required anymore
        if ( _prevTask ) {
          _prevTask.reset(nullptr);
        }

        _myTask->operator()(std::move(resStore));
      }

      std::unique_ptr<Prev> _prevTask;
      std::unique_ptr<AOp> _myTask;
    };

```



# Drawbacks
[drawbacks]: #drawbacks

Why should we **not** do this?

  * obscure corner cases
  * will it impact performance?
  * what other parts of the product will be affected?
  * will the solution be hard to maintain in the future?

# Alternatives
[alternatives]: #alternatives

- What other designs/options have been considered?
- What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

- What are the unknowns?
- What can happen if Murphy's law holds true?
