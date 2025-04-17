# Conclusions and Future Work

The maintenance burden of modern computing systems continues to grow with added complexity. Fortunately, tooling exists to drastically reduce that burden for many development and deployment tasks, and implementing such modern practices is well worth the effort for most distributed code projects.

Our efforts to modernize our `qhist` tool will ensure that future developments are easy to deploy, more robust, and less prone to manual process error. These improvements are largely self-documenting as well, so in the event that new maintainers take over the project, the spin-up time should be shorter.

There are, of course, many avenues for future work. With respect to `qhist` specifically, we would like to pursue the following directions, organized by time horizon:

**Short Term**
* Add more test coverage to the *pytest* suite
* Add more comments and docstrings in the code, where necessary
* Prototype extensions to the `PbsRecord` class, either via child classes or config-file defined custom fields
* Deploy the new `qhist` version to our users

**Medium Term**
* Support other distribution platforms (e.g., Conda, Spack)
* Implement a client-server mode, to resolve data locality issue

**Possible Long Term Directions**
* Support user-defined output fields derived from standard fields
* Support more statistical summaries (currently the user can display an average of numerical fields across selected jobs)
* Reintroduce some sorting capability without exploding memory requirements

Finally, we intend to apply this same modernization process to our other custom utilities like `qstat-cache` and `pbsprio`.