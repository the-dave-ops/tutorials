# K6 load testing
### Introdoction:
k6 load testing is a performance testing tool used to evaluate how well a system can handle a specific workload. It simulates multiple users accessing the system concurrently to assess its capacity and performance under stress. With k6, developers can identify bottlenecks, measure response times, and ensure their applications are robust and scalable. It offers scripting capabilities for crafting complex scenarios and generating realistic user behavior, making it a powerful tool for optimizing system performance.

K6 is a CLI based tool that you run from the terminal. it takes a javaScript testing file, and runs it again and again. you can configure how many VU (virtual users) will run. for example:
```
// Define options for the k6 test script
export const options = {
    // Define stages for ramping up virtual users over time
    stages: [
        // Stage 1: Ramp up from 0 to 3 virtual users over 20 seconds
        { duration: '20s', target: 3 },

        // Stage 2: Ramp up from 3 to 10 virtual users over 30 seconds
        { duration: '30s', target: 10 },

        // Stage 3: Ramp up from 10 to 20 virtual users over 1 minute and 30 seconds
        { duration: '1m30s', target: 20 },
    ],
};
```

To run the test, you run in the terminal:
```
k6 run script.js
```
the script will run, and will output the results of the test.

> To create a K6 test file from a swagger or a Postman collection, see this:
https://k6.io/docs/testing-guides/api-load-testing/#integrate-with-api-tools
{.is-info}



## Installation
The best way to use k6 is to build the binary with extentions inside a docker container, and use that binary (see example later).
K6 has a lot of extentions you can add to it. here is a list:
https://k6.io/docs/extensions/get-started/explore/
Some of the extentions are exporters that send the results of the test to other programs that collect data, and allow you to show the results in a dashboard.

the easyist way to see a dashboard with the results is to use the xk6-dashboard extention. 
we will build a binary of K6 with the extention in it, and then run the script and see the results in the browser.

## How to build the binary

Here is a example of a code that will build a K6 binary with the xk6-dashboard extention (and with other extentions as well):
```
docker run --rm -it -u "$(id -u):$(id -g)" -v "${PWD}:/xk6" grafana/xk6 build v0.43.1 \
  --with github.com/mostafa/xk6-kafka@v0.22.0 \
  --with github.com/grafana/xk6-dashboard@v0.7.2 \
  --with github.com/grafana/xk6-output-influxdb@v0.3.0
```
> Note that you will need to change the `@<version>` in this code to the updated one, if you want the updated versions of k6 and the exentions.
you can also use `@latest` to get the latest version.
{.is-info}


Running this code will create a `K6` binary in the current directory.

## How to see the results dashboard
After you build the K6 binary, you run it this way:
```
./k6 run --out web-dashboard script.js
```
here is a examle of the output:
![screenshot_from_2024-03-07_10-46-31.png](/screenshot_from_2024-03-07_10-46-31.png)

When the scritp runs it creates a HTML file with the results. (it update's in real time during the running of the script!)
you can go to `http://127.0.0.1:5665` and see them:
![screenshot_from_2024-03-06_17-09-45.png](/screenshot_from_2024-03-06_17-09-45.png)

> What's cool about this, that when the script is done you can save this page as a HTML file, and show it later.
That way you can run this script during a Gitlab CI, and save the file to the gitlab repo, with this command:
`./k6 run --out web-dashboard=report=test.html script.js`
this way you can save a copy of the test for each version of the app, in it's repo. NICE!
{.is-success}

![screenshot_from_2024-03-07_11-09-08.png](/screenshot_from_2024-03-07_11-09-08.png)

### code example of a gitlab-ci.yml with a K6 test:
```
stages:
  - test
k6_test:
  stage: test
  tags: 
    - shell
  script:
    #--------------------------------------------
    # run the k6 script
    #--------------------------------------------
    - date +"%d-%m-%Y---%H:%M:%S" > timestamp.txt
    - TIMESTAMP=$(cat timestamp.txt)
    - chmod +x k6
    - ./k6 run --out web-dashboard=report=$TIMESTAMP.html simple-test.js
    - mv $TIMESTAMP.html test-results/$TIMESTAMP.html
    #--------------------------------------------
    # Add the HTML file to the Git repository
    #--------------------------------------------
    - git config user.email "david@linnovate.net"
    - git config user.name "ci-bot"
    - git remote add k6 https://oauth2:${GIT_COMMIT_TOKEN}@gitlab.getapp.sh/getapp/k6-system-check.git || true
    - git add test-results/$TIMESTAMP.html
    - git commit -am "push file test-results/$TIMESTAMP.html"
    - git push k6 HEAD:master -o ci.skip

  only:
    - master
```