# Phase 7.0: MLOps

In this section you'll learn how to automate the model lifecycle from training to model deployment. 
We will in addition look at some additional considerations such as data and model tests.

## Phase 7.1: Create our first GitHub Action workflow
The first thing we want to do is to create a simple GitHub Action workflow which will be triggered to train a new model when a commit is pushed to our repository.

To do so, navigate to your forked repo and click on the `Actions` tab
On the page that appears, go ahead and select to set up a new .NET Core workflow
![action](https://github.com/aslotte/mlnet-workshop/blob/master/labs/media/action-dotnet-core-workflow.PNG)

GitHub will provide you with a template workflow that is intended to restore, build and test a .NET Core app. 

Replace the workflow file with the following content and commit it to your master branch.

```
name: .NET Core

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]
  
jobs:
  build:

    runs-on: ubuntu-latest

    steps:        
    - uses: actions/checkout@v2
    - name: Setup .NET Core
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.1.101   
    - name: Install dependencies
      run: dotnet restore src/MLNETWorkshop.sln
    - name: Build
      run: dotnet build src/MLNETWorkshop.sln --configuration Release --no-restore
``` 

If all goes well, a successful build should complete in less than a minute.

## Phase 7.2: Set up our data source
Great work, we're now able to compile our training project as part of our CI pipeline to ensure the integrity of our system. The next step is to automatically kick-off the training of our machine learning model. Before we can do that do we need to address a challenge we will face, which is the location of our training data. So far in this workshop you've had your training data available on disk, or as part of the GitHub repository. However, in many cases the training data is of 1-100 Gb large which makes it non-feasible to store it in GitHub. 

One way to solve this problem is to upload our data to an Azure FileShare and mount the fileshare on our Ubuntu build agent as part of each build so that the training application can access it. An Azure FileShare can handle concurrent load meaning that multiple build agents can read the data simultaneously. 

In order to achieve this we need to do two things:
1) Change the path to our data
2) Mount the fileshare as part of our Github workflow

### Change the path to our data
Navigate to the `Program.cs` file and change the `TRAIN_DATA_PATH` variable to:
```
  private static string TRAIN_DATA_FILEPATH = @"/media/data/true_car_listings.csv";
```

In addition, we'll also need to change the path to where we store our model. To do so, change the `MODEL_FILEPATH` variable to
```
  private static string MODEL_FILEPATH = MLConfiguration.GetModelPath();
```

What this will do is to store the model on the fileshare as well, with a unique id matching the Git commit sha.
Commit the changes to your master branch and push the changes to your repo.

### Mount the fileshare as part of our GitHub workflow
To mount the fileshare as part of our workflow, add the following just before the `Install dependencies` step and commit the changes to your master branch.
```
 - name: 'Create mount points'
      run: 'sudo mkdir /media/data'
    - name: 'Map disk drive to Azure Files share folder'
      run: 'sudo mount -t cifs //ndcmelbourne.file.core.windows.net/data /media/data -o vers=3.0,username=ndcmelbourne,password=${{ secrets.STORAGEKEY }},dir_mode=0777,file_mode=0777'
```

To be able to mount the fileshare, we'll also need to add the access key to the Azure Storage Container as a secret.
To add a secret, navigate to the `Settings` tab and select `Secrets` in the left menu:

![secrets](https://github.com/aslotte/mlnet-workshop/blob/master/labs/media/secrets.PNG)

Click on `New Secret` and add a new secret with the name of `STORAGEKEY`. The value will be provided to you by the facilitators of the workshop.

## Phase 7.3: Add model training to our GitHub workflow
In order to automatically train our model, we'll need to use the `dotnet run` command to run our console application.
To do so, go ahead and add the following to your GitHub Action's workflow file:
```
    - name: Train
      working-directory: 'src/TrainConsole'
      run: dotnet run --project TrainConsole.csproj
```

Your workflow file should now look as below. The GitHub action should take a couple of minutes to complete, and if all is setup correctly be successful. 

```
name: .NET Core

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]
  
jobs:
  build:

    runs-on: ubuntu-latest

    steps:        
    - uses: actions/checkout@v2
    - name: Setup .NET Core
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.1.101   
    - name: 'Create mount points'
      run: 'sudo mkdir /media/data'
    - name: 'Map disk drive to Azure Files share folder'
      run: 'sudo mount -t cifs //ndcmelbourne.file.core.windows.net/data /media/data -o vers=3.0,username=ndcmelbourne,password=${{ secrets.STORAGEKEY }},dir_mode=0777,file_mode=0777'
    - name: Install dependencies
      run: dotnet restore src/MLNETWorkshop.sln
    - name: Build
      run: dotnet build src/MLNETWorkshop.sln --configuration Release --no-restore
    - name: Train
      working-directory: 'src/TrainConsole'
      run: dotnet run --project TrainConsole.csproj 
```

## Phase 7.4: Data and Model Tests
Well done! If you've made it this far, you've succesfully setup a workflow that automatically trains your model on new commits. However, as with any well architected software application, we also require automated tests to be run to ensure that the application works as expected. Similarly we can add tests to our model training workflow. 

There are two types of tests that we will be looking into today:

1. **Data validation tests** to ensure the integrity of our training data 
2. **Model tests** to validate the quality of our trained model

### Data validation tests
To train our model, we use a dataset that consists of a number of features, such as price, the year the car was made, milage and so forth. To ensure the quality of our model, it's important to validate that the data is sound. We may for example want to verify that the dataset does not contain any negative numbers or other invalid data points. 

At our disposal, we have a `DataValidationsTests.cs` test class in the `DataTests` project (located under the `Tests` folder in the solutions explorer). 
This test class contains a number of tests that we will implement that will ultimately verify our dataset.

The first thing we would like to do is to set the correct path to our data (which will be located on the Azure FileShare as mentioned in previous steps).
Replace the `TRAIN_DATA_FILEPATH` variable located in `DataValidationTests` with

```
  private static string TRAIN_DATA_FILEPATH = @"/media/data/true_car_listings.csv";
```

The next thing we would like to do is to fill out the `Initalize` method. This method will be used to load the data using the given path and convert all rows to an enumerable of `ModelInput` rows. Change your `Initialize` method to below. Please note the `Rows` private member variable that will be used in the tests.

```
        private static IEnumerable<ModelInput> Rows;
        
        [ClassInitialize]
        public static void Initialize(TestContext testContext)
        {
            var mlContext = new MLContext();
            var data = mlContext.Data.LoadFromTextFile<ModelInput>(TRAIN_DATA_FILEPATH, hasHeader: true, separatorChar: ',');

            Rows = mlContext.Data.CreateEnumerable<ModelInput>(data, false);
        }
```

With the `Initialize` method setup, we're ready to start implementing our tests. 
Let's do two together, and then let's see if you're able to implement the other two yourself.

To verify a valid year range and that we don't have any negative prices in our dataset we can implement `VerifyValidPrice()` and `VerifyValidYear()` as follows:

```
        [TestMethod]
        public void VerifyValidPrice()
        {
            var hasNegativePrice = Rows.Any(x => x.Price < 0);

            hasNegativePrice.Should().BeFalse();
        }

        [TestMethod]
        public void VerifyValidYear()
        {
            var hasValidYears = Rows.All(x => x.Year > 1950 && x.Year < DateTime.Now.Year + 1);

            hasValidYears.Should().BeTrue();
        }
```

Based on these tests, take a couple of minutes and see if you can implement `VerifyValidMilage()` and `VerifyMinimumNumberOfRows()` yourself (assume that we need at least 10,000 rows).

The final two tests should look something like this:

```
        [TestMethod]
        public void VerifyValidMilage()
        {
            var hasInvalidMilage = Rows.Any(x => x.Mileage < 0);

            hasInvalidMilage.Should().BeFalse();
        }
        
        [TestMethod]
        public void VerifyMinimumNumberOfRows()
        {
            var rowCount = Rows.Count();

            rowCount.Should().BeGreaterThan(10000);
        }        
```

Excellent work so far. Go ahead and commit your changes and push them to your fork either by using a tool of your choice such as GitHub Desktop, Visual Studio or the Git CLI.
The next step is to add our data tests to our CI workflow to make sure they are run before the model training.

Do do so, open up the `dotnet-core.yml` file under `.github/workflows` and add the following just prior to `Train` step:

```
    - name: Data Tests
      working-directory: 'test/DataTests'     
      run: dotnet test DataTests.csproj
```

Your workflow file should now look as follows:

```
name: .NET Core

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]
  
jobs:
  build:

    runs-on: ubuntu-latest

    steps:        
    - uses: actions/checkout@v2
    - name: Setup .NET Core
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.1.101   
    - name: 'Create mount points'
      run: 'sudo mkdir /media/data'
    - name: 'Map disk drive to Azure Files share folder'
      run: 'sudo mount -t cifs //ndcmelbourne.file.core.windows.net/data /media/data -o vers=3.0,username=ndcmelbourne,password=${{ secrets.STORAGEKEY }},dir_mode=0777,file_mode=0777'
    - name: Install dependencies
      run: dotnet restore src/MLNETWorkshop.sln
    - name: Build
      run: dotnet build src/MLNETWorkshop.sln --configuration Release --no-restore
    - name: Train
      working-directory: 'src/TrainConsole'
      run: dotnet run --project TrainConsole.csproj 
    - name: Data Tests
      working-directory: 'test/DataTests'     
      run: dotnet test DataTests.csproj      
```

Commit your changes and push them to GitHub. This should kick of the workflow under the `Actions` tab and within 3-5 min you should seee a successful build if all goes well.

### Model tests
Brilliant, we're now able to run data validation tests as part of our workflow, but what our model tests? Let's have a look.
The model tests will run after we've trained our model in order to do some basic health checks. In more advanced scenarios, one may want to also compare the trained model to an existing model in production at this stage so that we can quickly determine if the model is worth investing additional time in or not.

At our disposal we have the `ModelTests.cs` test class in the `ModelTests` project (located under the `Tests` folder in the solution).



## Phase 7.5 Deployment - upload our model as an artifact
TBD - upload as an artifact

## Phase 7.6 - Consume our model 

