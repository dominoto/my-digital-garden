---
{"dg-publish":true,"permalink":"/resources/local-http-server-using-python/","created":"2024-03-02","updated":"2025-10-12"}
---

## How to run a local HTTP server using Python to check your website functionality easily

**Topics:** ==[[Web Development\|Web Development]],[[Python\|Python]]==

If you’re working on simple HTML/CSS/JS websites and need to test certain functionalities that don’t operate correctly when running the `.html` file directly (like cookies), here’s a straightforward method to set up your own local HTTP server:

1. Download and install the latest version of [Python](https://www.python.org/downloads/).
2. Navigate to the folder containing your website files and open the terminal (or Command Prompt) in that location.
    - If you’re using Windows, you can do this by holding down the `Shift` key, right-clicking on an empty space within the folder, and selecting `Open in Terminal`.
3. In the terminal, execute the command `python -m http.server`.
4. Launch any web browser and enter `http://localhost:8000/` in the address bar.
5. If everything is set up correctly, you should be able to view your website.