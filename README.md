# Computer Aided Validation Lab: Coding Style Guides

These style guides define the coding conventions used across the Computer Aided Validation Laboratory organisation. They are intended to make our software easier to review, maintain, optimise, and use across different projects and programming languages.

Our software is designed around three equally important principles:

1. **Make it correct.**
2. **Make it fast.**
3. **Make it simple for users.**

These principles should be considered together from the beginning of a design. Performance is not something we add only after the code works, and usability is not something we add only after the implementation is complete. Data layout, memory access, allocation, parallelism, numerical behaviour, API design, and failure modes should all be considered while the software is being designed.

The following style guides should be followed when writing code for the Computer Aided Validation Laboratory organisation:

* [Python Style Guide](guides/python_style_guide .md)

If you are writing performant compiled code for scientific or engineering simulation software start here:

* [Performance Oriented Software Guide](guides/performance_oriented_software_guide.md)

Then go to the language specific guide:

* [C++ Style Guide](guides/cpp_style_guide.md)
* [Zig Style Guide](guides/zig_style_guide.md)
