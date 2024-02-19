---
{"dg-publish":true,"permalink":"/digital-garden/deploying-a-react-app-with-git-hub-pages-react-router-and-vite-overcoming-deployment-challenges/","created":"2024-02-13","updated":"2024-02-14"}
---

![](https://cdn-images-1.medium.com/max/800/1*xUdfKiWBFsJ3N9h7yKUBFA.png)

Deploying a React app involves several considerations, and when incorporating tools like React Router and Vite, additional challenges can arise. In this short (but sweet) article, I’ll share my experiences deploying a React app using GitHub Pages, focusing on overcoming a specific problem encountered during the process: getting a **blank page** on deployed site even though everything worked fine in development.

### Deployment Challenges

### 1. Configuring `package.json`

One crucial step in deploying a React app to GitHub Pages is configuring the `package.json` file. The `homepage` field must be set to the GitHub Pages URL to ensure proper routing. Additionally, defining the `"predeploy"` and `"deploy"` scripts is essential for building and deploying the app.

```json
{  
    "homepage": "http://githubusername.github.io/vehicle-app",  
    "private": false,  
    "scripts": {  
        "predeploy": "npm run build",  
        "deploy": "gh-pages -d dist"  
    }  
}
```

### 2. Adapting React Router for GitHub Pages

Modifying the `main.jsx` file (or however the file with your Router structure is called) is necessary to accommodate GitHub Pages' subdirectory structure. Routes should be adjusted to include the app name from the GitHub Pages URL.

```js
const router = createBrowserRouter([  
    {  
        path: '/vehicle-app', // Append the app name from GitHub page to root location (/)  
        element: <Root />,  
        errorElement: <ErrorPage />,  
        children: [  
            {  
                errorElement: <ErrorPage />,  
                children: [  
                    {  
                        path: '/vehicle-app/', // Append the app name from GitHub page  
                        element: <MakeListPage />,  
                    },  
                    // ... other routes  
                ],  
            },  
        ],  
    },  
]);
```

### 3. Navigating GitHub Pages with React Router

In the `MakeTable.jsx` file, ensure proper navigation on GitHub Pages by appending the app name to route paths when using the `navigate` function.

```js
const onRowClicked = (event) => {  
    const selectedMake = event.data.Id;  
    vehicleMakeStore.selectedVehicleMake = event.data;  
    // Route to models list for the selected make  
    navigate(`/vehicle-app/vehicleMakes/${selectedMake}`); // Prepend the app name from GitHub page  
};
```

### Lessons Learned

Reflecting on the challenges faced during deployment, it became evident that meticulous configuration and adaptation were necessary. Understanding the intricacies of GitHub Pages, React Router, and Vite played a crucial role in overcoming these challenges. Just kidding, I just googled a lot :)

### Conclusion

Deploying a React app to GitHub Pages involves more than a simple `npm deploy`. By addressing specific challenges related to configuration and routing, you can ensure a smooth deployment process. Remember to adapt these solutions to your own projects, and explore further resources to enhance your deployment skills.