
- Before:
```cs

        public void UpdateCreditOption(int userId, int applicationId, int creditOptionId, CreditOptionDTOPost creditOptionDto)
        {

            var application = this.Application;


            if (application.Status == ApplicationStatus.ApprovalInProgress
                || application.Status == ApplicationStatus.Preparation
                || application.Status == ApplicationStatus.PendingAmendment
                || application.Status == ApplicationStatus.Submitted)
            {
                var credit = this.Reposiotry.GetCredit(application.CreditId);             

                if (credit.ApprovedOptionId == command.CreditOptionId && this.Reposiotry.ApplicationHasContract(command.ApplicationId))
                {
                   throw new Exception($"Can't update approved credit option: {creditOptionId} for application: {applicationId} when application has contract");                  
                }

                var existingCreditOption = this.Reposiotry.GetCreditOption(command.CreditOptionId);

                //existing credit option is marked as obsolete and new credit option is added
                existingCreditOption.Obsolete = true;
                existingCreditOption.ObsoleteDateTime = DateTime.UtcNow;

                var newCreditOption = new CreditOption();
                this._creditMapper.Map(command.CreditOption, newCreditOption);

                newCreditOption.UserId = command.UpdatedBy.UserId;
                newCreditOption.ParentCreditOptionId = existingCreditOption.ParentCreditOptionId ?? existingCreditOption.Id;
                newCreditOption.CreatedDateTime = existingCreditOption.CreatedDateTime;
                newCreditOption.ModifiedDateTime = DateTime.UtcNow;
                newCreditOption.ModifiedCreditOptionId = existingCreditOption.Id;
                newCreditOption.Type = existingCreditOption.Type;

                var rates = _platformInfo.InsuranceRates;
                double currentRate = rates[0]; //rate_1_60
                if (newCreditOption.Duration > 60)
                {
                    if (newCreditOption.Duration < 85)
                    {
                        currentRate = rates[1]; //rate_61_84
                    }
                    else if (newCreditOption.Duration >= 85 && newCreditOption.Duration <= 120)
                    {
                        currentRate = rates[2]; //rate_85_120
                    }
                }

                newCreditOption.InsuranceRate = currentRate;

                newCreditOption.MonthlyPayment = CreditCalculations.MonthlyPayment(newCreditOption.EffectiveInterestRate, newCreditOption.Amount, newCreditOption.Duration);
                newCreditOption.MonthlyInsurancePayment = CreditCalculations.MonthlyInsurancePayment(newCreditOption.EffectiveInterestRate, newCreditOption.Amount, newCreditOption.Duration, newCreditOption.InsuranceRate.Value);

                credit.Options.Add(newCreditOption);

                var savedItemsCount = this.Reposiotry.Save();
                if (savedItemsCount == 0) return;

                var update = false;
                //update initial option
                if (existingCreditOption.Type == CreditOptionType.Initial)
                {
                    credit.InitialOptionId = newCreditOption.Id;
                    update = true;
                }
                //update approved option - reset
                if (existingCreditOption.Id == credit.ApprovedOptionId)
                {
                    credit.ApprovedOptionId = null;
                    credit.ApprovedUserId = null;
                    update = true;
                }
                if (update)
                {
                    savedItemsCount = this.Reposiotry.Save();

                    if (savedItemsCount == 0) return;
                }
            }
        }
    }
```

- 1. Step. Introduce Param Object:

```cs
 public class UpdateCreditOptionCommand
    {
        public IUserInfo UpdatedBy { get; set; }
        public CreditOptionDTOPost CreditOption { get; set; }
        public int ApplicationId { get; set; }
        public int CreditOptionId { get; set; }
    }
```

- 2. Step. Apply clean code principles and read as a book:

```cs
 public void UpdateCreditOption(UpdateCreditOptionCommand command)
        {
            if (Application.CanUpdateCreditOption)
            {
                var credit = ObtainCredit();
                var existingCreditOption = ObtainExistingCreditOption(command);

                if (HasNoApprovedContract(command, credit))
                {
                    existingCreditOption.MarkAsObsolete();

                    credit.AddNewOption(BuildNewOption(command, existingCreditOption));

                    credit.UnApprove(existingCreditOption);

                    Persist();
                }
            }
        }

```

Some private helper for the method above

```cs
        #region Update Credit Option helpers

        private CreditOption ObtainExistingCreditOption(UpdateCreditOptionCommand command)
        {
            return this.Reposiotry.GetCreditOption(command.CreditOptionId);
        }

        private Credit ObtainCredit()
        {
            return this.Reposiotry.GetCredit(Application.CreditId);
        }

        private double ObtainRate(CreditOption newCreditOption)
        {
            var rates = _platformInfo.InsuranceRates;
            double currentRate = rates[0]; //rate_1_60
            if (newCreditOption.Duration > 60)
            {
                if (newCreditOption.Duration < 85)
                {
                    currentRate = rates[1]; //rate_61_84
                }
                else if (newCreditOption.Duration >= 85 && newCreditOption.Duration <= 120)
                {
                    currentRate = rates[2]; //rate_85_120
                }
            }

            return currentRate;
        }

        private CreditOption BuildNewOption(UpdateCreditOptionCommand command, CreditOption existingCreditOption)
        {
            var newCreditOption = new CreditOption();
            this._creditMapper.Map(command.CreditOption, newCreditOption);

            newCreditOption.UserId = command.UpdatedBy.UserId;
            newCreditOption.ParentCreditOptionId = existingCreditOption.ParentCreditOptionId ?? existingCreditOption.Id;
            newCreditOption.CreatedDateTime = existingCreditOption.CreatedDateTime;
            newCreditOption.ModifiedDateTime = DateTime.UtcNow;
            newCreditOption.ModifiedCreditOptionId = existingCreditOption.Id;
            newCreditOption.Type = existingCreditOption.Type;

            newCreditOption.InsuranceRate = ObtainRate(newCreditOption);

            newCreditOption.MonthlyPayment = CreditCalculations.MonthlyPayment(newCreditOption.EffectiveInterestRate, newCreditOption.Amount, newCreditOption.Duration);
            newCreditOption.MonthlyInsurancePayment = CreditCalculations.MonthlyInsurancePayment(newCreditOption.EffectiveInterestRate, newCreditOption.Amount, newCreditOption.Duration, newCreditOption.InsuranceRate.Value);


            return newCreditOption;
        }

        private bool HasNoApprovedContract(UpdateCreditOptionCommand command, Credit credit)
        {
            return !(credit.ApprovedOptionId == command.CreditOptionId && Application.HasContracts);
        }
        
        #endregion
```

- 3. Step. HasNoApprovedContract looks like a helper. Let's move the logic to the Applicaton class:

```cs

public class Application
    {
        ...

        public bool CanUpdateCreditOption => Status.OneOf(ApplicationStatus.ApprovalInProgress, ApplicationStatus.Preparation, ApplicationStatus.PendingAmendment, ApplicationStatus.Submitted);

        public bool HasNoApprovedContract(int creditOptionId)
        {
            return !(this.Credit.ApprovedOptionId == creditOptionId && HasContracts);
        }
    }
```

```cs
 public void UpdateCreditOption(UpdateCreditOptionCommand command)
        {
            if (Application.CanUpdateCreditOption)
            {
                var credit = ObtainCredit();
                var existingCreditOption = ObtainExistingCreditOption(command);

                if (this.Application.HasNoApprovedContract(command.CreditOptionId))
                {
                    existingCreditOption.MarkAsObsolete();

                    credit.AddNewOption(BuildNewOption(command, existingCreditOption));

                    credit.UnApprove(existingCreditOption);

                    this.Persist();
                }
            }
        }
```

Credit options is responsible for rate calculation now

```cs

  public class CreditOption
    {
        public int Id { get; set; }
        
        ...
        public void MarkAsObsolete()
        {
            this.Obsolete = true;
            this.ObsoleteDateTime = DateTime.UtcNow;
        }

        public void ApplyInsuranceRate(double[] availableRates)
        {
            var rate = availableRates[0];

            if (this.Duration > 60)
            {
                if (this.Duration < 85)
                {
                    rate = availableRates[1]; //rate_61_84
                }
                else if (this.Duration >= 85 && this.Duration <= 120)
                {
                    rate = availableRates[2]; //rate_85_120
                }
            }


            this.InsuranceRate = rate;
        }
    }
```

