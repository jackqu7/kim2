Process:

Decide feature is relevant

Document api and provide usage examples

Dev and testing

Review

Release


Use Pipeline objects instead of simple lists, making it easier to amend them.

Pipelines contain steps/groups: get_data, validate_data, transform_data, output_data etc

You only add something to one of the steps, so you can avoid having massive duplication if you want to change a minor thing.


Plural for inputters/outputters = pipeline operators?


Some sort of optparse type thing for pipeline operations?

