![GitFyle.Core](https://raw.githubusercontent.com/The-Standard-Organization/GitFyle.Core.Api/main/Resources/Images/gitfyle-gitlogo.png)

# POC: External Validation Engine
This is a POC to see if we can move the validation engine to an external library.  My hope is that is could be part of the Standard Framework Library. 

#What is different:
- The validation engine is now in a separate library and all the logic is fully tested on the validation engine to detect tampering.
  - Test to see that validation error will be throw for any amount of failing rules to detect exclusion logic around the ThrowIfContainsErrors
  - Test to check that a data item will be added to the exception data dictionary for every failing rule.  This will detect exclusion logic inside the foreach
  - We now test that a parameter name is provided in the validation criteria (InvalidOperationException)
  - We now test that a rule is provided (InvalidOperationException)
  - We now test that a message is present in the rule (InvalidOperationException)

- - The external validation engine is now surfaced via the `ValidationBroker.cs` in the API project and you can pass your rules in a similar way we did before.  You just provide the exception to throw, the message and the rules...
    - I have played here with support for local rules and external rules that hopefully over time will support the most of the common comparisons.
    - I realize this is bringing in external logic directly into our business domain, but my thinking is that if this becomes a framework library that it would be allowed similar to how we would use `System.Math`.  This would save duplication on testing the same rules on n services.  
    - In the below picture we can see that most of the rules is external and only the `IsInvalidUrl`  is a local rule with its own tests

![image](https://github.com/user-attachments/assets/f8079546-5ab3-4a70-99a1-1b4c49ac63f9)

- Since we now use the validation broker, the validation tests will need to change.  We need to do a mock setup for the validation broker to ensure an exception is thrown and we need to verify that the validation broker has been called with the same rules as in the implementation.  We do that by passing the rule collection where the `Condition` is set to match the expectation.   This will test that the rules in the implementation has reached the same expectation.

```cs
        [Theory]
        [InlineData(null)]
        [InlineData("")]
        [InlineData(" ")]
        [InlineData("   ")]
        [InlineData("\t")]
        [InlineData("\n")]
        [InlineData("\r")]
        [InlineData("\t \n\r")]
        public async Task ShouldCallValidateWithTheseExpectedValidationRulesAsync(string invalidString)
        {
            // given
            var invalidSource = new Source
            {
                Id = Guid.Empty,
                Name = invalidString,
                Url = "invalidString",
                CreatedBy = invalidString,
                UpdatedBy = invalidString,
                CreatedDate = default,
                UpdatedDate = default
            };

            (dynamic Rule, string Parameter)[] validationCriteria = CreateValidationCriteria(
                    source: invalidSource,
                    idCondition: true,
                    nameCondition: true,
                    urlCondition: true,
                    createdByCondition: true,
                    updatedByCondition: true,
                    createdDateCondition: true,
                    updatedDateCondition: true);

            var invalidSourceException = new InvalidSourceException(
                message: "Source is invalid, fix the errors and try again.");

            var expectedSourceValidationException =
                new SourceValidationException(
                    message: "Source validation error occurred, fix errors and try again.",
                    innerException: invalidSourceException);

            this.validationBrokerMock
                .Setup(broker =>
                    broker.Validate<InvalidSourceException>(
                        "Source is invalid, fix the errors and try again.",
                        It.IsAny<(dynamic Rule, string Parameter)[]>()))
                .ThrowsAsync(invalidSourceException);

            // when
            ValueTask<Source> addSourceTask =
                this.sourceService.AddSourceAsync(invalidSource);

            SourceValidationException actualSourceValidationException =
                await Assert.ThrowsAsync<SourceValidationException>(
                    addSourceTask.AsTask);

            // then
            actualSourceValidationException.Should().BeEquivalentTo(
                expectedSourceValidationException);

            this.validationBrokerMock.Verify(broker =>
                broker.Validate<InvalidSourceException>(
                    "Source is invalid, fix the errors and try again.",
                        It.Is(Validations.Comparer.SameRulesAs(validationCriteria))),
                            Times.Once);

            this.loggingBrokerMock.Verify(broker =>
                broker.LogError(It.Is(
                    SameExceptionAs(expectedSourceValidationException))),
                        Times.Once);

            this.storageBrokerMock.Verify(broker =>
                broker.InsertSourceAsync(It.IsAny<Source>()),
                    Times.Never);

            this.validationBrokerMock.VerifyNoOtherCalls();
            this.loggingBrokerMock.VerifyNoOtherCalls();
            this.storageBrokerMock.VerifyNoOtherCalls();
            this.dateTimeBrokerMock.VerifyNoOtherCalls();
        }
```
- All validation rules is also now specifically tested.  It now has two tests to support a valid and invalid outcome
![image](https://github.com/user-attachments/assets/c6c6f4a4-911c-40f2-85d1-b2bf2faf8cc9)
- Validation rules have also been extended to hold the values passed to the function used for the condition so that we can test if the values in the test match up with the implementation.  This is to address the 4th gap in our testing.
```cs
        internal static dynamic IsNotSameAs(
           string first,
           string second,
           string secondName) => new
           {
               Condition = first != second,
               Message = $"Text is not the same as {secondName}",
               Values = new object[] { first, second, secondName }
           };
```

**The above can be further extended with the normal exception handling but that is beyond the scope of this POC**
 
 